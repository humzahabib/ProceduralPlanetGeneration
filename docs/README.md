# Procedural Planet Renderer
### Summer Project Plan — June to August (12 Weeks)

---

## Goal

Build a planet with procedural terrain and a physically-based atmosphere in OpenGL/C++. By end of summer: a portfolio piece strong enough to support a master's application.

**Setup:** 4–5 hrs/day · June–August · one course + one project running in parallel · Vulkan as a slow-burn side track (30% of time).

---

## Glossary

| Term | Plain-English Meaning |
|---|---|
| **Cube-sphere** | Start with a cube (6 flat faces). Push every corner outward until the whole surface is a sphere. Better than a normal sphere because the triangles are distributed more evenly. |
| **Face / face grid** | One of the 6 sides of the starting cube. Each face is a grid of small squares (quads) that together tile that side. |
| **Quad / quad grid** | A quad is a rectangle made of 2 triangles. A quad grid = many quads in rows and columns, like graph paper. |
| **Vertex / vertex shader** | A vertex is a corner point in 3D space. A vertex shader is a tiny GPU program that runs once per corner and can move it. |
| **Fragment shader** | GPU program that runs once per pixel. Decides the final color — used here to paint terrain by elevation (blue=ocean, green=land, etc.). |
| **FBM noise** | Fractal Brownian Motion. Stack several layers of random noise, each smaller and finer than the last. The result looks like natural terrain. |
| **MVP matrix** | Model-View-Projection. Three transforms combined: where the object sits in the world, where the camera is, and the lens/projection. |
| **FBO / framebuffer object** | An off-screen render target. Instead of drawing to the screen, you draw into a texture, then use that texture in the next pass. |
| **Multipass rendering** | Render the scene more than once. Each pass reads the previous pass's texture and adds an effect (e.g. blur for bloom). |
| **LUT (look-up table)** | A pre-baked texture that stores expensive calculation results. At runtime you just sample the texture instead of re-doing the math. |
| **Transmittance** | How much sunlight survives after passing through a column of atmosphere. Stored in a LUT to avoid recomputing it every frame. |
| **Scattering** | Light bouncing off air molecules. Rayleigh scattering (tiny molecules) makes sky blue. Mie scattering (dust/haze) makes sunsets orange. |
| **Optical depth** | The total amount of atmosphere a light ray passes through — affects how much scattering happens. |
| **Atmospheric limb** | The glowing halo you see around a planet's edge in space. The signature visual of a good atmosphere shader. |
| **Quadtree LOD** | Split each face into 4 sub-squares when the camera is close; merge them back when far. Keeps triangle count manageable. |
| **T-junction** | A crack that appears at the seam between a high-detail and low-detail region. Must be fixed with stitching geometry. |
| **Log depth buffer** | A trick to store depth values logarithmically so the GPU can handle the huge range from 1 metre to 1000 km without z-fighting. |
| **Z-fighting** | Two surfaces at nearly the same depth flicker because the GPU cannot tell which is in front. Log depth buffer fixes this. |
| **ImGui** | Dear ImGui — a simple C++ library for live sliders and buttons overlaid on your OpenGL window. Lets you tweak noise params in real time. |
| **VAO / VBO** | Vertex Array Object / Vertex Buffer Object. OpenGL structures that hold your mesh data on the GPU. |
| **Descriptor set (Vulkan)** | Vulkan's equivalent of uniform variables — how you pass textures and buffers to shaders. |
| **Render pass (Vulkan)** | Vulkan's explicit declaration of what you're drawing into and what should happen to it before/after. |

---

## Phase 1 — The Mesh (Weeks 1–5)

Before anything looks like a planet you need the right shape. This phase builds the cube-sphere — the foundation every later feature sits on. Uses skills you already have (VAOs, MVP matrices). Main new idea: how to organize 6 faces, and how normalization turns a cube into a sphere.

### Phase 1A — One Face (Days 1–4)

Forget the sphere for now. Draw one flat square made of a grid of smaller squares. Exactly like the quad you'd use for a fullscreen effect, just tiled N×N times. Upload vertices and indices to the GPU once, draw with `glDrawElements`.

| Task | Duration | What You Build | Key Concept |
|---|---|---|---|
| Create N×N quad grid | 2 days | A flat grey square on screen, N sliders of resolution via ImGui | Index buffer layout, strip vs list topology |
| Add per-vertex normals | 1 day | Flat shading on the quad responds to a directional light | Normal vectors, how lighting uses them |
| Wireframe toggle | 1 day | Press W to see the triangle edges — confirms grid density | `glPolygonMode`, debug views |

### Phase 1B — All 6 Faces + Sphere (Days 5–10)

Duplicate the face 6 times, each rotated to face a different direction (top, bottom, front, back, left, right). Then normalize every vertex position — divide by its length so all points land on a sphere of radius 1. Triangles near corners will be denser than at face centers. Acceptable for now.

| Task | Duration | What You Build | Key Concept |
|---|---|---|---|
| Define 6 face orientation matrices | 2 days | Six flat squares forming a cube in the scene | Rotation matrices, scene hierarchy |
| Normalize vertices onto sphere | 1 day | Cube collapses into a smooth sphere | `normalize(v) = v / length(v)` |
| Smooth normals after normalization | 1 day | Lighting looks round, not faceted | Normals must be re-derived post-warp |
| Plug into solar system scene | 2 days | Sphere orbits the sun alongside existing planets | Scene graph reuse, scale considerations |

> **Optional:** Equal-area correction for corner triangle density. Only matters visually at very high resolutions — skip and revisit only if you see obvious pinching artifacts.

---

## Phase 2 — Terrain (Weeks 3–5, overlaps Phase 1)

Once you have a sphere, push terrain out of it. Do this on the GPU in the vertex shader so every frame the displacement is essentially free. Feed a 3D position into a noise function, get back a height value, multiply by a small scale, move the vertex outward along its normal.

| Task | Duration | What You Build | Key Concept |
|---|---|---|---|
| GPU Perlin noise in GLSL | 2 days | Noisy sphere with bumpy surface | 3D noise, hash functions in GLSL |
| FBM — stack 6 octaves of noise | 2 days | Continents, mountains, fine detail all visible | Lacunarity (frequency), gain (amplitude) |
| Elevation color ramp | 1 day | Ocean blue / land green / mountain grey / snow white | `mix()`, `smoothstep()`, colour lerping |
| ImGui live tweaking panel | 1 day | Sliders change terrain shape in real time | Uniform uploads, feedback loop workflow |

> **Why 3D noise instead of a 2D texture?** A 2D texture wrapped onto a sphere always has a seam where the edges meet. 3D noise takes the (x,y,z) position on the sphere directly — no seam possible.

---

## Week 6 — Buffer / Catch-up Week

Intentionally empty. Course and parallel project will spike. If nothing slipped, use this week to polish ImGui controls, write dev-log notes, and start reading the Bruneton paper.

---

## Phase 3 — Atmosphere (Weeks 7–11)

The centrepiece of the project and the hardest phase. Target paper: **Bruneton & Neyret 2008** ("Precomputed Atmospheric Scattering"). Simulates how sunlight bounces through air to produce blue sky, orange sunsets, and the glowing halo at a planet's edge. You do not implement this from scratch — Bruneton published a cleaned-up open-source implementation on GitHub that you adapt.

### Phase 3A — Bridge Exercises (Week 7)

Two small projects that teach the building blocks. Do both before touching the atmosphere code.

| Task | Duration | What You Build | Key Concept |
|---|---|---|---|
| Sunset shader on a fullscreen quad | 3 days | Gradient sky that shifts from blue to orange as sun angle changes | Optical depth integral, Rayleigh phase function, FBOs |
| Bloom post-process effect | 2 days | Bright areas glow — required for realistic sun and atmospheric limb | Multipass rendering, ping-pong FBOs, gaussian blur |

### Phase 3B — Bruneton Atmosphere (Weeks 8–11)

The Bruneton model pre-bakes expensive light-transport calculations into textures (LUTs) at startup. At runtime the shaders just sample those textures, so it runs in real time. Build the LUTs one at a time in this order — each depends on the previous.

| Task | Duration | What You Build | Key Concept |
|---|---|---|---|
| Read paper + annotate Bruneton repo | 4 days | Annotated source code, personal notes on each LUT | What transmittance, single scatter, and irradiance mean |
| Transmittance LUT | 4 days | Texture showing how much sunlight reaches each altitude | Numerical integration along a ray through the atmosphere |
| Single-scatter LUT | 5 days | Sky colour from one bounce of sunlight off air molecules | In-scatter integral, Rayleigh vs Mie phase functions |
| Multiple-scatter LUT + full sky dome | 5 days | Realistic sky with correct horizon glow and limb lighting | Irradiance table, iterative multi-bounce accumulation |
| Integrate with planet, add bloom | 3 days | Planet inside its atmosphere, sun creates a lens-flare halo | Depth testing planet vs sky, HDR + tone-mapping |

> ⚠️ **Known stuck point:** After implementing the LUTs the sky often renders completely black or completely white. Usually a coordinate-space mismatch (kilometres vs metres) or a wrong exponent in the density falloff. Budget a full frustration week for this. It is normal.

---

## Week 12 — Portfolio Week (No New Features)

Record a screen capture. Push to GitHub with a README. Write a 500-word project description. Take screenshots for your master's application portfolio. **This week is as important as any coding week.**

---

## Phase 4 — Quadtree LOD (Stretch Goal)

LOD = Level of Detail. When the camera is close to the planet surface, you need more triangles for fine detail. When far away, fewer triangles are fine. A quadtree achieves this: each face square can split into 4 smaller squares or merge back, based on distance to camera.

| Task | Duration | What You Build | Key Concept |
|---|---|---|---|
| Quadtree data structure per face | 3 days | CPU-side tree that decides split/merge each frame | Recursive data structures, screen-space error metric |
| Dynamic mesh rebuild from tree | 3 days | GPU mesh updates as camera moves — detail appears near surface | VBO streaming, partial updates |
| T-junction stitching | 3 days | No cracks at LOD seams | Skirts or explicit neighbour-aware edge vertices |
| Log depth buffer | 1 day | No z-fighting between terrain and atmosphere at any distance | `gl_FragDepth`, Outerra logarithmic formula (10 lines) |

> LOD is genuinely hard and will not be missed from the portfolio if your atmosphere looks good. Prioritise a polished Phase 3 over a rushed Phase 4.

---

## Vulkan Parallel Track (30% of Daily Time, No Deadline Pressure)

Vulkan is a lower-level GPU API than OpenGL. It forces you to be explicit about things OpenGL hides (memory, synchronisation, render passes). Learning it alongside OpenGL — not instead of it — gives you the vocabulary for any serious graphics job or master's programme.

| Task | Duration | What You Build | Key Concept |
|---|---|---|---|
| Vulkan triangle (vulkan-tutorial.com) | Weeks 1–4 | A triangle on screen — harder than it sounds in Vulkan | Swap chain, render pass, pipeline state object, command buffers |
| Conceptual mapping to OpenGL | Week 4 | Written notes: what Vulkan concept = what OpenGL call | Why Vulkan is explicit where OpenGL was implicit |
| Port cube-sphere mesh to Vulkan | Weeks 9–12 | Same sphere rendered via Vulkan pipeline | Descriptor sets (= uniforms), vertex buffers, synchronisation |

---

## Key Deadlines

| Week | Checkpoint | Deliverable |
|---|---|---|
| **Week 5** | Planet renders in solar system scene | Cube-sphere with FBM terrain, elevation colours, ImGui sliders. Normals correct. Orbits sun. |
| **Week 6** | Buffer checkpoint | If anything slipped in weeks 1–5, catch up now. Otherwise read ahead into Bruneton. |
| **Week 7** | Bridge exercises complete | Sunset shader working. Bloom working. You understand FBOs and optical depth. |
| **Week 11** | Full atmosphere working | Blue sky, orange sunset, atmospheric limb glow visible from space. |
| **Week 12** | Portfolio ready | GitHub repo, screen recording, written project description, screenshots. |

---

## Resources

| Resource | What It's For | When to Use |
|---|---|---|
| learnopengl.com — Advanced Lighting | Physically-based lighting, normal maps, HDR, bloom | Weeks 1–7 |
| vulkan-tutorial.com | Step-by-step Vulkan from scratch | Weeks 1–4 (30% time) |
| Bruneton & Neyret 2008 paper (free, Inria website) | The atmosphere math — read for understanding | Week 7 |
| Eric Bruneton's GitHub repo (atmosphere model) | The actual code you adapt — read alongside the paper | Weeks 7–11 |
| Sebastian Lague YouTube — Procedural Planets | Visual reference and motivation (Unity/C#, you use C++/OpenGL) | Anytime |
| 3Blue1Brown — Essence of Linear Algebra | Refresh matrix intuition if anything feels shaky | As needed |
| Dear ImGui (GitHub) | The live-tweaking overlay library | Week 4 onward |
| GLM library docs | Maths types (`vec3`, `mat4`) and functions you'll use constantly | Throughout |

---

> **Keep a dev log.** A plain markdown file where you write one paragraph per day: what you tried, what broke, what fixed it. Makes debugging faster, reinforces learning, and is raw material for your master's application statement of purpose.
