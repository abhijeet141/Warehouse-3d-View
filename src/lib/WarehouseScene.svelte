<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
  import Stats from 'three/examples/jsm/libs/stats.module.js';
  import type { Segment, SegmentType } from '../types';

  export let segments: Segment[];
  export let visibleTypes: Set<SegmentType>;
  export let camera: {
    position: [number, number, number];
    fov?: number;
    near?: number;
    far?: number;
  } = { position: [180000, 130000, 280000], near: 100, far: 800000 };
  export let orbit: {
    target: [number, number, number];
    minDistance: number;
    maxDistance: number;
  } = { target: [53700, 7825, 43525], minDistance: 2000, maxDistance: 400000 };
  export let grid: {
    size: number;
    cellSize: number;
    sectionSize: number;
    centerXZ: [number, number];
    fadeDistance?: number;
    fadeStrength?: number;
  } = { size: 300000, cellSize: 2000, sectionSize: 10000, centerXZ: [53700, 43525], fadeDistance: 400000, fadeStrength: 1 };
  export let floorY = 100;
  export let axesSize = 10000;
  export let originSphereRadius = 500;

  const COLOR_MAP: Record<SegmentType, number> = {
    AISLE: 0x3b82f6,
    BAY:   0xf97316,
    LEVEL: 0x22c55e,
    SPACE: 0xef4444,
  };

  const OPACITY_MAP: Record<SegmentType, number> = {
    AISLE: 0.06,
    BAY:   0.15,
    LEVEL: 0.30,
    SPACE: 0.15,
  };

  // Fixed draw order for the transparent tiers. Without this three.js sorts
  // them by camera distance per-object (not per-instance), so at low camera
  // angles the red SPACE mesh can be drawn after BAY/LEVEL and tint the racks.
  const TIER_ORDER: Record<SegmentType, number> = { AISLE: 0, SPACE: 0, BAY: 1, LEVEL: 2 };

  const SPACE_HOVER_COLOR   = 0x7f1d1d;
  const SPACE_HOVER_OPACITY = 0.95;

  const FIND_COLOR = 0xfacc15;

  let findGroup: THREE.Group | null = null;
  let findMatchSegs: Segment[] = [];
  let findQuery = '';
  let findStatus = '';
  let findStatusKind: 'ok' | 'err' = 'ok';
  let findInputEl: HTMLInputElement;
  let suggestions: Segment[] = [];
  let suggestionsOpen = false;
  let activeSuggestion = -1;

  const TYPE_HEX: Record<SegmentType, string> = {
    AISLE: '#3b82f6',
    BAY:   '#f97316',
    LEVEL: '#22c55e',
    SPACE: '#ef4444',
  };

  let container: HTMLDivElement;
  let canvas: HTMLCanvasElement;
  let tooltipX = 0;
  let tooltipY = 0;
  let hoverInfo: {
    fullName: string;
    type: SegmentType;
    coords: [number, number, number];
    dims:   [number, number, number];
  } | null = null;

  let scene: THREE.Scene;
  let perspectiveCamera: THREE.PerspectiveCamera;
  let renderer: THREE.WebGLRenderer;
  let controls: OrbitControls;
  let raycaster: THREE.Raycaster;
  let pointer: THREE.Vector2;
  let stats: Stats;
  let rafId = 0;

  const containerGroups = new Map<SegmentType, THREE.Group>();
  const tierInstancedMeshes = new Map<SegmentType, THREE.InstancedMesh>();
  const tierEdgeMeshes = new Map<SegmentType, THREE.LineSegments>();
  let spaceSegments: Segment[] = [];
  let highlightGroup: THREE.Group | null = null;
  let highlightedId: number | null = null;
  let gridMesh: THREE.Mesh | null = null;

  // Procedural anti-aliased floor grid (port of drei's <Grid>).
  // Renders one plane; the fragment shader draws cell + section lines and fades with distance.
  function makeShaderGrid(): THREE.Mesh {
    const planeGeom = new THREE.PlaneGeometry(grid.size, grid.size);
    planeGeom.rotateX(-Math.PI / 2);

    const mat = new THREE.ShaderMaterial({
      uniforms: {
        uCellSize:         { value: grid.cellSize },
        uSectionSize:      { value: grid.sectionSize },
        uCellColor:        { value: new THREE.Color(0x4b5563) },
        uSectionColor:     { value: new THREE.Color(0xcbd5e1) },
        uCellThickness:    { value: 1.0 },
        uSectionThickness: { value: 1.8 },
        uFadeDistance:     { value: grid.fadeDistance ?? 400000 },
        uFadeStrength:     { value: grid.fadeStrength ?? 1.0 },
        uCameraPos:        { value: new THREE.Vector3() },
      },
      vertexShader: `
        varying vec3 vWorldPos;
        void main() {
          vec4 wp = modelMatrix * vec4(position, 1.0);
          vWorldPos = wp.xyz;
          gl_Position = projectionMatrix * viewMatrix * wp;
        }
      `,
      fragmentShader: `
        uniform float uCellSize;
        uniform float uSectionSize;
        uniform vec3  uCellColor;
        uniform vec3  uSectionColor;
        uniform float uCellThickness;
        uniform float uSectionThickness;
        uniform float uFadeDistance;
        uniform float uFadeStrength;
        uniform vec3  uCameraPos;
        varying vec3 vWorldPos;

        float gridLine(vec2 coord, float thickness) {
          vec2 g = abs(fract(coord - 0.5) - 0.5) / fwidth(coord);
          float line = min(g.x, g.y);
          return 1.0 - min(line / thickness, 1.0);
        }

        void main() {
          vec2 xz = vWorldPos.xz;
          float cell    = gridLine(xz / uCellSize,    uCellThickness);
          float section = gridLine(xz / uSectionSize, uSectionThickness);

          vec3 color = mix(uCellColor, uSectionColor, section);
          float alpha = max(cell * 0.55, section);

          float dist = distance(uCameraPos.xz, xz);
          float fade = 1.0 - smoothstep(uFadeDistance * 0.5, uFadeDistance, dist);
          alpha *= pow(fade, uFadeStrength);

          if (alpha < 0.01) discard;
          gl_FragColor = vec4(color, alpha);
        }
      `,
      transparent: true,
      side: THREE.DoubleSide,
      depthWrite: false,
    });

    const mesh = new THREE.Mesh(planeGeom, mat);
    mesh.position.set(grid.centerXZ[0], -1, grid.centerXZ[1]);
    mesh.renderOrder = -1;
    mesh.raycast = () => {};
    return mesh;
  }

  function makeContainerMesh(seg: Segment): THREE.Group {
    const group = new THREE.Group();
    const geom = new THREE.BoxGeometry(seg.dimensionX, seg.dimensionZ, seg.dimensionY);
    const mat = new THREE.MeshStandardMaterial({
      color: COLOR_MAP[seg.type],
      transparent: true,
      opacity: OPACITY_MAP[seg.type],
      depthWrite: false,
    });
    const mesh = new THREE.Mesh(geom, mat);
    mesh.position.set(
      seg.coordinateX + seg.dimensionX / 2,
      seg.coordinateZ + seg.dimensionZ / 2,
      seg.coordinateY + seg.dimensionY / 2,
    );
    mesh.raycast = () => {};
    group.add(mesh);

    const edgeGeom = new THREE.EdgesGeometry(geom);
    const edgeMat = new THREE.LineBasicMaterial({ color: COLOR_MAP[seg.type] });
    const edges = new THREE.LineSegments(edgeGeom, edgeMat);
    edges.position.copy(mesh.position);
    edges.scale.set(1.001, 1.001, 1.001);
    group.add(edges);

    return group;
  }

  // Build one InstancedMesh that draws every segment of a tier in a single draw call.
  function buildTierInstancedMesh(list: Segment[], type: SegmentType): THREE.InstancedMesh {
    const geom = new THREE.BoxGeometry(1, 1, 1);
    const mat = new THREE.MeshStandardMaterial({
      color: COLOR_MAP[type],
      transparent: true,
      opacity: OPACITY_MAP[type],
      depthWrite: false,
    });
    const inst = new THREE.InstancedMesh(geom, mat, list.length);
    const dummy = new THREE.Object3D();
    for (let i = 0; i < list.length; i++) {
      const s = list[i];
      dummy.position.set(
        s.coordinateX + s.dimensionX / 2,
        s.coordinateZ + s.dimensionZ / 2,
        s.coordinateY + s.dimensionY / 2,
      );
      dummy.scale.set(s.dimensionX, s.dimensionZ, s.dimensionY);
      dummy.rotation.set(0, 0, 0);
      dummy.updateMatrix();
      inst.setMatrixAt(i, dummy.matrix);
    }
    inst.instanceMatrix.needsUpdate = true;
    inst.computeBoundingSphere();
    inst.computeBoundingBox();
    // Containers (BAY/LEVEL) shouldn't intercept pointer events; only SPACE does.
    if (type !== 'SPACE') inst.raycast = () => {};
    return inst;
  }

  // Build a single LineSegments containing every cube's 12 edges (pre-transformed),
  // so one draw call covers all edge outlines for the tier.
  function buildTierEdgeMesh(list: Segment[], type: SegmentType): THREE.LineSegments {
    // 12 edges × 2 endpoints = 24 vertices per cube
    const unitEdges = new THREE.EdgesGeometry(new THREE.BoxGeometry(1, 1, 1));
    const unitPositions = unitEdges.getAttribute('position') as THREE.BufferAttribute;
    const vertsPerCube = unitPositions.count;
    const allPositions = new Float32Array(list.length * vertsPerCube * 3);

    const v = new THREE.Vector3();
    for (let i = 0; i < list.length; i++) {
      const s = list[i];
      const cx = s.coordinateX + s.dimensionX / 2;
      const cy = s.coordinateZ + s.dimensionZ / 2;
      const cz = s.coordinateY + s.dimensionY / 2;
      const sx = s.dimensionX;
      const sy = s.dimensionZ;
      const sz = s.dimensionY;
      const offset = i * vertsPerCube * 3;
      for (let j = 0; j < vertsPerCube; j++) {
        v.fromBufferAttribute(unitPositions, j);
        allPositions[offset + j * 3 + 0] = v.x * sx + cx;
        allPositions[offset + j * 3 + 1] = v.y * sy + cy;
        allPositions[offset + j * 3 + 2] = v.z * sz + cz;
      }
    }
    unitEdges.dispose();

    const geom = new THREE.BufferGeometry();
    geom.setAttribute('position', new THREE.BufferAttribute(allPositions, 3));
    geom.computeBoundingSphere();
    geom.computeBoundingBox();

    const mat = new THREE.LineBasicMaterial({ color: COLOR_MAP[type] });
    const lines = new THREE.LineSegments(geom, mat);
    lines.raycast = () => {};
    return lines;
  }

  // outlineOnly: used when the segment is already shown as a yellow find match —
  // the red fill would fight with the yellow box, so only the wireframe is drawn.
  function makeHighlightGroup(seg: Segment, outlineOnly = false): THREE.Group {
    const group = new THREE.Group();
    const geom = new THREE.BoxGeometry(seg.dimensionX, seg.dimensionZ, seg.dimensionY);
    const center = new THREE.Vector3(
      seg.coordinateX + seg.dimensionX / 2,
      seg.coordinateZ + seg.dimensionZ / 2,
      seg.coordinateY + seg.dimensionY / 2,
    );

    if (!outlineOnly) {
      const mat = new THREE.MeshStandardMaterial({
        color: SPACE_HOVER_COLOR,
        transparent: true,
        opacity: SPACE_HOVER_OPACITY,
        depthWrite: false,
      });
      const mesh = new THREE.Mesh(geom, mat);
      mesh.position.copy(center);
      mesh.renderOrder = 5; // above the tier boxes (0–2)
      mesh.raycast = () => {};
      group.add(mesh);
    }

    const edgeGeom = new THREE.EdgesGeometry(geom);
    const edgeMat = new THREE.LineBasicMaterial({ color: SPACE_HOVER_COLOR });
    const edges = new THREE.LineSegments(edgeGeom, edgeMat);
    edges.position.copy(center);
    // Outline-only sits on top of the find box (scale 1.02), so push it out
    // a bit further to stay visible.
    const s = outlineOnly ? 1.04 : 1.001;
    edges.scale.set(s, s, s);
    edges.renderOrder = outlineOnly ? 7 : 5; // above find boxes (6) / tier boxes (0–2)
    group.add(edges);

    return group;
  }

  // True when the segment sits inside (or is) one of the yellow find boxes —
  // there the red hover fill would fight with the yellow, so only the
  // wireframe outline is drawn.
  function isInsideFind(seg: Segment): boolean {
    if (findMatchSegs.length === 0) return false;
    const cx = seg.coordinateX + seg.dimensionX / 2;
    const cy = seg.coordinateY + seg.dimensionY / 2;
    const cz = seg.coordinateZ + seg.dimensionZ / 2;
    return findMatchSegs.some((m) =>
      cx >= m.coordinateX && cx <= m.coordinateX + m.dimensionX &&
      cy >= m.coordinateY && cy <= m.coordinateY + m.dimensionY &&
      cz >= m.coordinateZ && cz <= m.coordinateZ + m.dimensionZ,
    );
  }

  function clearFind() {
    findMatchSegs = [];
    if (!findGroup) return;
    scene.remove(findGroup);
    findGroup.traverse((o) => {
      if (o instanceof THREE.Mesh || o instanceof THREE.LineSegments) {
        o.geometry.dispose();
        const m = o.material as THREE.Material | THREE.Material[];
        Array.isArray(m) ? m.forEach((x) => x.dispose()) : m.dispose();
      }
    });
    findGroup = null;
  }

  function makeFindBox(seg: Segment): THREE.Group {
    const g = new THREE.Group();
    const geom = new THREE.BoxGeometry(seg.dimensionX, seg.dimensionZ, seg.dimensionY);
    const mesh = new THREE.Mesh(
      geom,
      new THREE.MeshStandardMaterial({
        color: FIND_COLOR, transparent: true, opacity: 0.85,
        depthWrite: false, side: THREE.DoubleSide,
      }),
    );
    mesh.position.set(
      seg.coordinateX + seg.dimensionX / 2,
      seg.coordinateZ + seg.dimensionZ / 2,
      seg.coordinateY + seg.dimensionY / 2,
    );
    mesh.renderOrder = 6; // above hover (5)
    mesh.raycast = () => {};
    g.add(mesh);

    const edges = new THREE.LineSegments(
      new THREE.EdgesGeometry(geom),
      new THREE.LineBasicMaterial({ color: FIND_COLOR }),
    );
    edges.position.copy(mesh.position);
    edges.scale.set(1.02, 1.02, 1.02);
    edges.renderOrder = 6;
    g.add(edges);
    return g;
  }

  // Never frame closer than this (mm) — keeps the camera outside the rack even
  // when the framed box itself is a single small leaf segment.
  const FIND_MIN_DIST = 12000;

  // Leaf segments (LEVEL/SPACE) are small boxes deep inside the rack; framing
  // them directly puts the camera inside the bay. Widen the framing box to the
  // containing BAY so a nested match gets the same overview a bay search does.
  function widenToBayContext(matches: Segment[], box: THREE.Box3): THREE.Box3 {
    const out = box.clone();
    if (matches.length > 50) return out; // many matches ⇒ box is already wide
    const bays = segments.filter((s) => s.type === 'BAY');
    for (const seg of matches) {
      if (seg.type !== 'LEVEL' && seg.type !== 'SPACE') continue;
      const cx = seg.coordinateX + seg.dimensionX / 2;
      const cy = seg.coordinateY + seg.dimensionY / 2;
      const cz = seg.coordinateZ + seg.dimensionZ / 2;
      const bay = bays.find((b) =>
        cx >= b.coordinateX && cx <= b.coordinateX + b.dimensionX &&
        cy >= b.coordinateY && cy <= b.coordinateY + b.dimensionY &&
        cz >= b.coordinateZ && cz <= b.coordinateZ + b.dimensionZ,
      );
      if (!bay) continue;
      out.expandByPoint(new THREE.Vector3(bay.coordinateX, bay.coordinateZ, bay.coordinateY));
      out.expandByPoint(new THREE.Vector3(
        bay.coordinateX + bay.dimensionX,
        bay.coordinateZ + bay.dimensionZ,
        bay.coordinateY + bay.dimensionY,
      ));
    }
    return out;
  }

  function frameBox(box: THREE.Box3) {
    const center = box.getCenter(new THREE.Vector3());
    const size = box.getSize(new THREE.Vector3());
    const maxDim = Math.max(size.x, size.y, size.z, 1);
    const fov = (perspectiveCamera.fov * Math.PI) / 180;
    let dist = (maxDim / 2) / Math.tan(fov / 2) * 2.5; // 2.5 = padding
    dist = Math.max(orbit.minDistance * 1.1, FIND_MIN_DIST, Math.min(orbit.maxDistance * 0.9, dist));

    const dir = new THREE.Vector3().subVectors(perspectiveCamera.position, controls.target);
    if (dir.lengthSq() < 1e-6) dir.set(0.6, 0.5, 0.8);
    dir.normalize();

    controls.target.copy(center);
    perspectiveCamera.position.copy(center).addScaledVector(dir, dist);
    controls.update();
  }

  function findLocation(query?: string) {
    if (query !== undefined) findQuery = query;
    suggestionsOpen = false;
    activeSuggestion = -1;
    clearFind();
    const q = findQuery.trim().toUpperCase();
    if (!q) { findStatus = ''; return; }

    const exact = segments.filter((s) => s.fullName.toUpperCase() === q);
    const matches = exact.length ? exact : segments.filter((s) => s.fullName.toUpperCase().startsWith(q));

    if (matches.length === 0) { findStatus = `"${findQuery}" not found`; findStatusKind = 'err'; return; }

    findGroup = new THREE.Group();
    const box = new THREE.Box3();
    for (const seg of matches) {
      findGroup.add(makeFindBox(seg));
      box.expandByPoint(new THREE.Vector3(seg.coordinateX, seg.coordinateZ, seg.coordinateY));
      box.expandByPoint(new THREE.Vector3(
        seg.coordinateX + seg.dimensionX,
        seg.coordinateZ + seg.dimensionZ,
        seg.coordinateY + seg.dimensionY,
      ));
    }
    scene.add(findGroup);
    findMatchSegs = matches;
    findStatus = matches.length === 1 ? `Found ${matches[0].fullName}` : `Found ${matches.length} matches`;
    findStatusKind = 'ok';
    frameBox(widenToBayContext(matches, box));
  }

  function updateSuggestions() {
    findStatus = '';
    const q = findQuery.trim().toUpperCase();
    if (!q) {
      suggestions = []; suggestionsOpen = false; activeSuggestion = -1;
      return;
    }
    const out: Segment[] = [];
    for (const s of segments) {
      if (s.fullName.toUpperCase().startsWith(q)) {
        out.push(s);
        if (out.length >= 8) break;
      }
    }
    suggestions = out;
    suggestionsOpen = out.length > 0;
    activeSuggestion = -1;
  }

  function clearSearch() {
    findQuery = '';
    findStatus = '';
    suggestions = [];
    suggestionsOpen = false;
    activeSuggestion = -1;
    clearFind();
    findInputEl?.focus();
  }

  function onFindKeydown(e: KeyboardEvent) {
    if (e.key === 'ArrowDown' && suggestionsOpen) {
      e.preventDefault();
      activeSuggestion = (activeSuggestion + 1) % suggestions.length;
    } else if (e.key === 'ArrowUp' && suggestionsOpen) {
      e.preventDefault();
      activeSuggestion = (activeSuggestion - 1 + suggestions.length) % suggestions.length;
    } else if (e.key === 'Enter') {
      if (suggestionsOpen && activeSuggestion >= 0) findLocation(suggestions[activeSuggestion].fullName);
      else findLocation();
    } else if (e.key === 'Escape') {
      if (suggestionsOpen) { suggestionsOpen = false; activeSuggestion = -1; }
      else clearSearch();
    }
  }

  // "/" anywhere on the page jumps to the search box (ignored while typing in a field).
  function onWindowKeydown(e: KeyboardEvent) {
    const t = e.target as HTMLElement | null;
    if (t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA')) return;
    if (e.key === '/') {
      e.preventDefault();
      findInputEl?.focus();
    }
  }

  $: qLen = findQuery.trim().length;

  function clearHighlight() {
    if (highlightGroup) {
      scene.remove(highlightGroup);
      highlightGroup.traverse((obj) => {
        if (obj instanceof THREE.Mesh || obj instanceof THREE.LineSegments) {
          obj.geometry.dispose();
          const mt = obj.material as THREE.Material | THREE.Material[];
          if (Array.isArray(mt)) mt.forEach((m) => m.dispose());
          else mt.dispose();
        }
      });
      highlightGroup = null;
    }
    highlightedId = null;
    hoverInfo = null;
  }

  function applyHighlight(instanceId: number) {
    if (instanceId === highlightedId) return;
    clearHighlight();
    const seg = spaceSegments[instanceId];
    if (!seg) return;
    highlightedId = instanceId;
    highlightGroup = makeHighlightGroup(seg, isInsideFind(seg));
    scene.add(highlightGroup);
    hoverInfo = {
      fullName: seg.fullName,
      type: seg.type,
      coords: [seg.coordinateX, seg.coordinateY, seg.coordinateZ],
      dims:   [seg.dimensionX, seg.dimensionY, seg.dimensionZ],
    };
  }

  function onPointerMove(event: PointerEvent) {
    const rect = canvas.getBoundingClientRect();
    pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    tooltipX = event.clientX - rect.left;
    tooltipY = event.clientY - rect.top;

    const spaceInst = tierInstancedMeshes.get('SPACE');
    if (!spaceInst || !visibleTypes.has('SPACE')) {
      clearHighlight();
      return;
    }
    raycaster.setFromCamera(pointer, perspectiveCamera);
    const hits = raycaster.intersectObject(spaceInst, false);
    if (hits.length > 0 && hits[0].instanceId !== undefined) {
      applyHighlight(hits[0].instanceId);
    } else {
      clearHighlight();
    }
  }

  function onPointerLeave() {
    clearHighlight();
  }

  function onResize() {
    if (!container) return;
    perspectiveCamera.aspect = container.clientWidth / container.clientHeight;
    perspectiveCamera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  }

  function applyVisibility() {
    for (const [type, group] of containerGroups) {
      group.visible = visibleTypes.has(type);
    }
    for (const [type, inst] of tierInstancedMeshes) {
      inst.visible = visibleTypes.has(type);
    }
    for (const [type, edges] of tierEdgeMeshes) {
      edges.visible = visibleTypes.has(type);
    }
    if (!visibleTypes.has('SPACE')) clearHighlight();
  }

  $: if (scene && visibleTypes) applyVisibility();

  function setupScene() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0f172a);

    perspectiveCamera = new THREE.PerspectiveCamera(
      camera.fov ?? 50,
      container.clientWidth / container.clientHeight,
      camera.near ?? 100,
      camera.far ?? 800000,
    );
    perspectiveCamera.position.set(...camera.position);

    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(container.clientWidth, container.clientHeight);

    controls = new OrbitControls(perspectiveCamera, canvas);
    controls.target.set(...orbit.target);
    controls.minDistance = orbit.minDistance;
    controls.maxDistance = orbit.maxDistance;
    // Unrestricted rotation; the floor-Y clamp in the animate loop keeps the
    // camera position itself from dipping below the floor, while still
    // allowing low/zoomed-in viewing angles.
    controls.maxPolarAngle = Math.PI;
    controls.update();

    scene.add(new THREE.AmbientLight(0xffffff, 0.55));
    const dirLight = new THREE.DirectionalLight(0xffffff, 0.9);
    dirLight.position.set(30000, 40000, 20000);
    scene.add(dirLight);
    const pointLight = new THREE.PointLight(0xffffff, 0.3);
    pointLight.position.set(-10000, 15000, -10000);
    scene.add(pointLight);

    gridMesh = makeShaderGrid();
    scene.add(gridMesh);

    const axes = new THREE.AxesHelper(axesSize);
    scene.add(axes);

    const originSphere = new THREE.Mesh(
      new THREE.SphereGeometry(originSphereRadius, 16, 16),
      new THREE.MeshBasicMaterial({ color: 0xff0000 }),
    );
    originSphere.position.set(0, originSphereRadius, 0);
    scene.add(originSphere);

    raycaster = new THREE.Raycaster();
    pointer = new THREE.Vector2();
    stats = new Stats();
    stats.dom.style.position = 'absolute';
    stats.dom.style.top = '4px';
    stats.dom.style.left = '4px';
    container.appendChild(stats.dom);

    populateSegments();
    applyVisibility();

    window.addEventListener('resize', onResize);
    window.addEventListener('keydown', onWindowKeydown);
    canvas.addEventListener('pointermove', onPointerMove);
    canvas.addEventListener('pointerleave', onPointerLeave);
    animate();
  }

  function populateSegments() {
    const buckets = new Map<SegmentType, Segment[]>();
    for (const seg of segments) {
      const list = buckets.get(seg.type) ?? [];
      list.push(seg);
      buckets.set(seg.type, list);
    }

    // AISLE stays as plain meshes (only 18 of them; cheap, and may want per-aisle labels later)
    const aisleList = buckets.get('AISLE') ?? [];
    if (aisleList.length > 0) {
      const group = new THREE.Group();
      group.name = 'AISLE';
      for (const seg of aisleList) group.add(makeContainerMesh(seg));
      containerGroups.set('AISLE', group);
      scene.add(group);
    }

    // BAY, LEVEL, SPACE all use InstancedMesh for huge draw-call savings.
    // BAY/LEVEL also get a single LineSegments edge pass to preserve outline look.
    for (const type of ['BAY', 'LEVEL', 'SPACE'] as const) {
      const list = buckets.get(type) ?? [];
      if (list.length === 0) continue;

      const inst = buildTierInstancedMesh(list, type);
      inst.name = type;
      inst.renderOrder = TIER_ORDER[type];
      tierInstancedMeshes.set(type, inst);
      scene.add(inst);

      if (type !== 'SPACE') {
        const edges = buildTierEdgeMesh(list, type);
        edges.name = `${type}_EDGES`;
        tierEdgeMeshes.set(type, edges);
        scene.add(edges);
      }
    }

    spaceSegments = buckets.get('SPACE') ?? [];
  }

  function animate() {
    rafId = requestAnimationFrame(animate);
    controls.update();
    // Hard floor: keep the camera position above the floor plane regardless of
    // rotation or zoom direction. Allows any tilt angle / zoom level otherwise.
    if (perspectiveCamera.position.y < floorY) {
      perspectiveCamera.position.y = floorY;
    }
    if (gridMesh) {
      const u = (gridMesh.material as THREE.ShaderMaterial).uniforms.uCameraPos.value as THREE.Vector3;
      u.copy(perspectiveCamera.position);
    }
    renderer.render(scene, perspectiveCamera);
    if (stats) stats.update();
  }

  onMount(() => {
    setupScene();
  });

  onDestroy(() => {
    cancelAnimationFrame(rafId);
    window.removeEventListener('resize', onResize);
    window.removeEventListener('keydown', onWindowKeydown);
    if (canvas) {
      canvas.removeEventListener('pointermove', onPointerMove);
      canvas.removeEventListener('pointerleave', onPointerLeave);
    }
    if (stats && stats.dom && stats.dom.parentElement) {
      stats.dom.parentElement.removeChild(stats.dom);
    }
    if (renderer) renderer.dispose();
  });
</script>

<div class="scene-container" bind:this={container}>
  <canvas bind:this={canvas}></canvas>
  {#if hoverInfo}
    <div class="tooltip" style="left: {tooltipX + 14}px; top: {tooltipY + 14}px;">
      <strong>{hoverInfo.fullName}</strong> ({hoverInfo.type})
      <br />
      coord: ({hoverInfo.coords[0]}, {hoverInfo.coords[1]}, {hoverInfo.coords[2]})
      <br />
      dim:&nbsp;&nbsp; ({hoverInfo.dims[0]} × {hoverInfo.dims[1]} × {hoverInfo.dims[2]})
    </div>
  {/if}
  <div class="find-ui">
    <div class="find-bar">
      <svg class="find-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round">
        <circle cx="11" cy="11" r="7" />
        <line x1="16.5" y1="16.5" x2="21" y2="21" />
      </svg>
      <input
        class="find-input" type="text" placeholder="Search location… e.g. N11G03 or A01"
        bind:this={findInputEl}
        bind:value={findQuery}
        on:input={updateSuggestions}
        on:keydown={onFindKeydown}
        on:focus={() => { if (suggestions.length) suggestionsOpen = true; }}
        on:blur={() => setTimeout(() => { suggestionsOpen = false; }, 120)}
        spellcheck="false" autocomplete="off"
      />
      {#if findQuery}
        <button class="find-clear" on:click={clearSearch} title="Clear search">✕</button>
      {:else}
        <kbd class="find-kbd">/</kbd>
      {/if}
      <button class="find-btn" on:click={() => findLocation()}>Find</button>
    </div>

    {#if suggestionsOpen}
      <ul class="find-suggestions">
        {#each suggestions as s, i}
          <li>
            <button
              class="suggestion" class:active={i === activeSuggestion}
              on:mousedown|preventDefault={() => findLocation(s.fullName)}
              on:mouseenter={() => (activeSuggestion = i)}
            >
              <span class="s-name"><strong>{s.fullName.slice(0, qLen)}</strong>{s.fullName.slice(qLen)}</span>
              <span class="s-type" style="--t: {TYPE_HEX[s.type]}">{s.type}</span>
            </button>
          </li>
        {/each}
      </ul>
    {/if}

    {#if findStatus}
      <div class="find-status" class:err={findStatusKind === 'err'}>
        {findStatus}
      </div>
    {/if}
  </div>
</div>

<style>
  .scene-container {
    position: relative;
    flex: 1;
    overflow: hidden;
    background: #0f172a;
  }
  canvas {
    display: block;
    width: 100%;
    height: 100%;
  }
  .tooltip {
    position: absolute;
    background: rgba(0, 0, 0, 0.85);
    color: white;
    padding: 6px 10px;
    border-radius: 4px;
    font-size: 12px;
    white-space: nowrap;
    pointer-events: none;
    font-family: monospace;
    z-index: 10;
  }
  .find-ui {
    position: absolute; top: 14px; left: 50%; transform: translateX(-50%);
    z-index: 10; display: flex; flex-direction: column; align-items: center; gap: 8px;
    width: min(440px, calc(100% - 32px));
  }
  .find-bar {
    display: flex; align-items: center; gap: 9px; width: 100%;
    background: rgba(11, 18, 32, 0.88);
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    border: 1.5px solid #334155;
    border-radius: 999px;
    padding: 6px 6px 6px 15px;
    box-shadow: 0 4px 24px rgba(0, 0, 0, 0.45);
    transition: border-color 0.15s, box-shadow 0.15s;
  }
  .find-bar:focus-within {
    border-color: #3b82f6;
    box-shadow: 0 4px 28px rgba(59, 130, 246, 0.28);
  }
  .find-icon { width: 15px; height: 15px; color: #64748b; flex: none; transition: color 0.15s; }
  .find-bar:focus-within .find-icon { color: #3b82f6; }
  .find-input {
    flex: 1; min-width: 0; background: transparent; border: none; outline: none;
    color: #e2e8f0; font-size: 13px; font-family: monospace; letter-spacing: 0.3px;
  }
  .find-input::placeholder { color: #475569; }
  .find-clear {
    flex: none; width: 22px; height: 22px; border-radius: 50%;
    border: none; background: #1e293b; color: #94a3b8; cursor: pointer;
    font-size: 11px; line-height: 1; display: grid; place-items: center;
    transition: background 0.15s, color 0.15s;
  }
  .find-clear:hover { background: #334155; color: #e2e8f0; }
  .find-kbd {
    flex: none; color: #475569; border: 1px solid #334155; border-radius: 4px;
    font-size: 10px; font-family: monospace; padding: 1px 6px;
  }
  .find-btn {
    flex: none; background: #2563eb; border: none; color: #fff;
    padding: 6px 18px; border-radius: 999px; cursor: pointer;
    font-size: 12px; font-weight: 600; letter-spacing: 0.3px;
    transition: background 0.15s;
  }
  .find-btn:hover { background: #3b82f6; }
  .find-btn:active { background: #1d4ed8; }
  .find-suggestions {
    width: 100%; margin: 0; padding: 6px; list-style: none;
    background: rgba(11, 18, 32, 0.95);
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    border: 1px solid #334155; border-radius: 14px;
    box-shadow: 0 12px 32px rgba(0, 0, 0, 0.5);
    max-height: 280px; overflow-y: auto;
  }
  .find-suggestions li { margin: 0; padding: 0; }
  .suggestion {
    width: 100%; display: flex; justify-content: space-between; align-items: center; gap: 12px;
    background: transparent; border: none; cursor: pointer;
    padding: 8px 11px; border-radius: 9px; color: #cbd5e1;
    font-family: monospace; font-size: 12px; text-align: left;
    transition: background 0.1s;
  }
  .suggestion.active { background: rgba(59, 130, 246, 0.16); color: #e2e8f0; }
  .s-name strong { color: #60a5fa; font-weight: 700; }
  .s-type {
    flex: none; font-size: 9px; padding: 2px 8px; border-radius: 999px;
    border: 1px solid var(--t); color: var(--t); letter-spacing: 0.6px;
  }
  .find-status {
    font-size: 11px; font-family: monospace; white-space: nowrap;
    padding: 4px 13px; border-radius: 999px;
    background: rgba(34, 197, 94, 0.12); color: #4ade80;
    border: 1px solid rgba(34, 197, 94, 0.35);
  }
  .find-status.err {
    background: rgba(239, 68, 68, 0.12); color: #f87171;
    border-color: rgba(239, 68, 68, 0.35);
  }
</style>
