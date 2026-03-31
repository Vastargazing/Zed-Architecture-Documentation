# Zed Architecture Documentation

<img width="1200" height="630" alt="image" src="https://github.com/user-attachments/assets/b8806c2e-0458-4f4f-9033-98602754ba98" />

A comprehensive guide to the Zed code editor's architecture, organized as a series of **architectural atoms** — self-contained documents that each explain one fundamental layer of the system.

---

## Reading Order

The atoms are ordered from the lowest-level foundation to the highest-level features. Each atom builds on concepts from previous ones.

| # | Atom | What It Covers |
|---|------|---------------|
| **1** | [GPUI](1_GPUI.md) | Custom GPU UI framework: entity system, rendering pipeline, actions, events, focus, platform abstraction |
| **2** | [Workspace & Pane](2_Workspace-Pane.md) | Window layout: split trees, tab management, dock panels, persistence, focus routing |
| **3** | [Project & Worktree](3_Project-Worktree.md) | File system layer: background scanning, snapshots, file watching, buffer store, dual local/remote architecture |
| **4** | [Editor & Buffer](4_Editor-Buffer.md) | Text editing engine: Rope data structure, display map pipeline (5 transform layers + CreaseMap), selections, anchors, undo/redo, syntax highlighting |
| **5** | [Settings & Registry](5_Settings-Registry.md) | Configuration system: hierarchical settings, keymap binding, settings traits, per-language/per-file overrides |
| **6** | [LSP & Language Server](6_LSP-Language-Server.md) | Editor intelligence: LSP client, server lifecycle, language registry, multi-server coordination |
| **7** | [Extensions](7_Extensions.md) | Plugin architecture: WASM sandboxing, Extension trait, delegates, extension index, hot reload |
| **8** | [Collaboration & CRDT](8_Collaboration-CRDT.md) | Real-time editing: CRDT, Lamport clocks, version vectors, RPC protocol, collab server |
| **9** | [Terminal & Tasks](9_Terminal-Tasks.md) | Execution environment: terminal emulation (Alacritty), PTY, task templates, variable substitution, runnables |
| **10** | [Agent & AI](10_Agent-AI.md) | AI assistance: model providers, tool system, inline assistant, context management, MCP |

---

## Architecture Overview

![zed_architecture_overview](https://github.com/user-attachments/assets/60ac6d93-14c1-41d0-b94c-98fd05dd6ca7)

![<svg width="100%" viewBox="0 0 680 620" xmlns="http://www.w3.org/2000/svg">
<defs>
  <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
    <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
  </marker>
<mask id="imagine-text-gaps-vn21rp" maskUnits="userSpaceOnUse"><rect x="0" y="0" width="680" height="620" fill="white"/><rect x="319.07415771484375" y="559.4141235351562" width="41.851680755615234" height="21.171789169311523" fill="black" rx="2"/><rect x="164.0674285888672" y="578.5588989257812" width="351.8650817871094" height="18.88221836090088" fill="black" rx="2"/><rect x="260.03271484375" y="499.41412353515625" width="159.9345703125" height="21.171789169311523" fill="black" rx="2"/><rect x="175.41690063476562" y="518.5588989257812" width="329.1661376953125" height="18.88221836090088" fill="black" rx="2"/><rect x="311.8924255371094" y="421.41412353515625" width="56.21516799926758" height="21.171789169311523" fill="black" rx="2"/><rect x="137.31253051757812" y="442.5589294433594" width="405.3749084472656" height="18.88221836090088" fill="black" rx="2"/><rect x="242.04258728027344" y="460.55889892578125" width="195.9148406982422" height="18.88221836090088" fill="black" rx="2"/><rect x="125.63533020019531" y="327.41412353515625" width="48.729339599609375" height="21.171789169311523" fill="black" rx="2"/><rect x="76.01153564453125" y="348.5589294433594" width="147.97691345214844" height="18.88221836090088" fill="black" rx="2"/><rect x="88.76963806152344" y="366.55889892578125" width="122.4607162475586" height="18.88221836090088" fill="black" rx="2"/><rect x="115.74813079833984" y="384.55889892578125" width="68.50372695922852" height="18.88221836090088" fill="black" rx="2"/><rect x="317.9393310546875" y="327.41412353515625" width="64.12134552001953" height="21.171789169311523" fill="black" rx="2"/><rect x="306.236572265625" y="348.5589294433594" width="87.52685546875" height="18.88221836090088" fill="black" rx="2"/><rect x="330.5096130371094" y="366.55889892578125" width="38.98077201843262" height="18.88221836090088" fill="black" rx="2"/><rect x="508.4178466796875" y="327.41412353515625" width="63.16437530517578" height="21.171789169311523" fill="black" rx="2"/><rect x="493.9470520019531" y="348.5589294433594" width="92.10599517822266" height="18.88221836090088" fill="black" rx="2"/><rect x="486.8055419921875" y="366.55889892578125" width="106.38899230957031" height="18.88221836090088" fill="black" rx="2"/><rect x="298.6558532714844" y="243.4141082763672" width="82.68833923339844" height="21.171789169311523" fill="black" rx="2"/><rect x="136.75802612304688" y="264.55889892578125" width="406.4839172363281" height="18.88221836090088" fill="black" rx="2"/><rect x="247.0107879638672" y="282.55889892578125" width="186.10882568359375" height="18.88221836090088" fill="black" rx="2"/><rect x="146.3568115234375" y="147.4141082763672" width="77.28638458251953" height="21.171789169311523" fill="black" rx="2"/><rect x="98.44573211669922" y="168.55889892578125" width="173.10855102539062" height="18.88221836090088" fill="black" rx="2"/><rect x="120.98371124267578" y="186.55889892578125" width="128.14341735839844" height="18.88221836090088" fill="black" rx="2"/><rect x="424.3072509765625" y="147.4141082763672" width="141.3854522705078" height="21.171789169311523" fill="black" rx="2"/><rect x="414.48712158203125" y="168.55889892578125" width="161.02569580078125" height="18.88221836090088" fill="black" rx="2"/><rect x="394.3460388183594" y="186.55889892578125" width="201.30784606933594" height="18.88221836090088" fill="black" rx="2"/><rect x="269.3475341796875" y="55.41410827636719" width="141.30496215820312" height="21.171785354614258" fill="black" rx="2"/><rect x="167.47048950195312" y="76.55889892578125" width="345.0589904785156" height="18.88221836090088" fill="black" rx="2"/><rect x="193.31312561035156" y="94.55889892578125" width="293.3736877441406" height="18.88221836090088" fill="black" rx="2"/></mask></defs>

<!-- Layer 1: GPUI -->
<g onclick="sendPrompt('Explain the GPUI entity system in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="40" y="550" width="600" height="48" rx="8" stroke-width="0.5" style="fill:rgb(68, 68, 65);stroke:rgb(180, 178, 169);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="340" y="570" text-anchor="middle" dominant-baseline="central" style="fill:rgb(211, 209, 199);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">GPUI</text>
  <text x="340" y="588" text-anchor="middle" dominant-baseline="central" style="fill:rgb(180, 178, 169);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Entity system · render pipeline · actions · platform abstraction</text>
</g>

<!-- Layer 2: Text/CRDT -->
<g onclick="sendPrompt('Explain the Rope and CRDT layer in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="40" y="490" width="600" height="48" rx="8" stroke-width="0.5" style="fill:rgb(8, 80, 65);stroke:rgb(93, 202, 165);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="340" y="510" text-anchor="middle" dominant-baseline="central" style="fill:rgb(159, 225, 203);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Text / Language / CRDT</text>
  <text x="340" y="528" text-anchor="middle" dominant-baseline="central" style="fill:rgb(93, 202, 165);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Rope (SumTree) · Fragments · Lamport clocks · Tree-sitter</text>
</g>

<!-- Layer 3: Project -->
<g onclick="sendPrompt('Explain the Project layer and its stores in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="40" y="410" width="600" height="68" rx="8" stroke-width="0.5" style="fill:rgb(12, 68, 124);stroke:rgb(133, 183, 235);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="340" y="432" text-anchor="middle" dominant-baseline="central" style="fill:rgb(181, 212, 244);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Project</text>
  <text x="340" y="452" text-anchor="middle" dominant-baseline="central" style="fill:rgb(133, 183, 235);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">WorktreeStore · BufferStore · LspStore · GitStore · TaskStore · DapStore</text>
  <text x="340" y="470" text-anchor="middle" dominant-baseline="central" style="fill:rgb(133, 183, 235);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">dual local / remote for every store</text>
</g>

<!-- Layer 4: Middle row — Editor, Terminal, Settings -->
<g onclick="sendPrompt('Explain the Editor and DisplayMap pipeline in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="40" y="310" width="220" height="88" rx="8" stroke-width="0.5" style="fill:rgb(60, 52, 137);stroke:rgb(175, 169, 236);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="150" y="338" text-anchor="middle" dominant-baseline="central" style="fill:rgb(206, 203, 246);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Editor</text>
  <text x="150" y="358" text-anchor="middle" dominant-baseline="central" style="fill:rgb(175, 169, 236);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">MultiBuffer · DisplayMap</text>
  <text x="150" y="376" text-anchor="middle" dominant-baseline="central" style="fill:rgb(175, 169, 236);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Selections · Anchors</text>
  <text x="150" y="394" text-anchor="middle" dominant-baseline="central" style="fill:rgb(175, 169, 236);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Undo/redo</text>
</g>

<g onclick="sendPrompt('Explain the Terminal and Tasks layer in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="270" y="310" width="160" height="88" rx="8" stroke-width="0.5" style="fill:rgb(99, 56, 6);stroke:rgb(239, 159, 39);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="350" y="338" text-anchor="middle" dominant-baseline="central" style="fill:rgb(250, 199, 117);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Terminal</text>
  <text x="350" y="358" text-anchor="middle" dominant-baseline="central" style="fill:rgb(239, 159, 39);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Alacritty · PTY</text>
  <text x="350" y="376" text-anchor="middle" dominant-baseline="central" style="fill:rgb(239, 159, 39);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Tasks</text>
</g>

<g onclick="sendPrompt('Explain the Settings and keymap system in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="440" y="310" width="200" height="88" rx="8" stroke-width="0.5" style="fill:rgb(113, 43, 19);stroke:rgb(240, 153, 123);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="540" y="338" text-anchor="middle" dominant-baseline="central" style="fill:rgb(245, 196, 179);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Settings</text>
  <text x="540" y="358" text-anchor="middle" dominant-baseline="central" style="fill:rgb(240, 153, 123);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Store · Keymap</text>
  <text x="540" y="376" text-anchor="middle" dominant-baseline="central" style="fill:rgb(240, 153, 123);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Scoped overrides</text>
</g>

<!-- Layer 5: Workspace -->
<g onclick="sendPrompt('Explain the Workspace and Pane layout system in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="40" y="230" width="600" height="68" rx="8" stroke-width="0.5" style="fill:rgb(12, 68, 124);stroke:rgb(133, 183, 235);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="340" y="254" text-anchor="middle" dominant-baseline="central" style="fill:rgb(181, 212, 244);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Workspace</text>
  <text x="340" y="274" text-anchor="middle" dominant-baseline="central" style="fill:rgb(133, 183, 235);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">PaneGroup (split tree) · Dock panels (left/right/bottom) · WorkspaceDb</text>
  <text x="340" y="292" text-anchor="middle" dominant-baseline="central" style="fill:rgb(133, 183, 235);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">focus routing · tab management</text>
</g>

<!-- Layer 6: Top row — Agent, Extensions -->
<g onclick="sendPrompt('Explain the Agent and AI system in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="40" y="130" width="290" height="88" rx="8" stroke-width="0.5" style="fill:rgb(60, 52, 137);stroke:rgb(175, 169, 236);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="185" y="158" text-anchor="middle" dominant-baseline="central" style="fill:rgb(206, 203, 246);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Agent (AI)</text>
  <text x="185" y="178" text-anchor="middle" dominant-baseline="central" style="fill:rgb(175, 169, 236);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Model providers · tool system</text>
  <text x="185" y="196" text-anchor="middle" dominant-baseline="central" style="fill:rgb(175, 169, 236);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Inline assistant · MCP</text>
</g>

<g onclick="sendPrompt('Explain the Extensions WASM system in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="350" y="130" width="290" height="88" rx="8" stroke-width="0.5" style="fill:rgb(8, 80, 65);stroke:rgb(93, 202, 165);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="495" y="158" text-anchor="middle" dominant-baseline="central" style="fill:rgb(159, 225, 203);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Extensions (WASM)</text>
  <text x="495" y="178" text-anchor="middle" dominant-baseline="central" style="fill:rgb(93, 202, 165);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Sandboxed · WIT interfaces</text>
  <text x="495" y="196" text-anchor="middle" dominant-baseline="central" style="fill:rgb(93, 202, 165);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Languages · themes · LSP adapters</text>
</g>

<!-- Collab — sidebar -->
<g onclick="sendPrompt('Explain the Collaboration RPC system in Zed')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
  <rect x="40" y="40" width="600" height="78" rx="8" stroke-width="0.5" style="fill:rgb(68, 68, 65);stroke:rgb(180, 178, 169);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="340" y="66" text-anchor="middle" dominant-baseline="central" style="fill:rgb(211, 209, 199);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Collaboration (RPC)</text>
  <text x="340" y="86" text-anchor="middle" dominant-baseline="central" style="fill:rgb(180, 178, 169);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">Protobuf over WebSocket · collab server (relay, not resolver)</text>
  <text x="340" y="104" text-anchor="middle" dominant-baseline="central" style="fill:rgb(180, 178, 169);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">dual-mode local / remote mirrors every store above</text>
</g>

<!-- Arrows between layers -->
<line x1="340" y1="228" x2="340" y2="300" marker-end="url(#arrow)" opacity="0.4" mask="url(#imagine-text-gaps-vn21rp)" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.4;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
<line x1="340" y1="408" x2="340" y2="400" marker-end="url(#arrow)" opacity="0.4" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.4;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
<line x1="340" y1="488" x2="340" y2="480" marker-end="url(#arrow)" opacity="0.4" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.4;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
<line x1="340" y1="548" x2="340" y2="540" marker-end="url(#arrow)" opacity="0.4" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.4;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
</svg>Uploading zed_architecture_overview.svg…]()


---

## Key Architectural Principles

### 1. Everything is an Entity

State lives in entities managed by GPUI's entity map. Access goes through typed handles (`Entity<T>`, `WeakEntity<T>`). No global mutable state — all mutations go through `entity.update(cx, ...)`.

### 2. Dual Local/Remote Architecture

Every data store (Worktree, Buffer, LSP) has two variants — Local (direct access) and Remote (RPC proxy). This enables seamless collaboration without special-casing.

### 3. Snapshot-Based Consistency

Background tasks (file scanning, syntax parsing, soft-wrapping) produce immutable snapshots. UI code reads snapshots for consistent, non-blocking access. Writers and readers never contend.

### 4. CRDT for Collaboration

Text buffers use a CRDT with Lamport clocks, ensuring all replicas converge to the same state regardless of operation order. The collab server acts as a relay for live project sessions (no conflict resolution); channel buffers are additionally persisted server-side in a database.

### 5. WASM Sandboxing for Extensions

Extensions run in WASM sandboxes with no direct filesystem or process access. They interact with Zed through WIT-exported interfaces (delegates, HTTP client, Node.js helpers). HTTP requests are allowed via a host-exported `fetch()`. This provides strong security isolation while keeping extensions capable.

### 6. SumTree Everywhere

Zed's custom SumTree (a B-tree with aggregate summaries) powers both Rope (text storage) and Worktree (file indexing). It provides O(log n) operations for all common editor operations.

### 7. Layered Display Pipeline

Text goes through 5 coordinate-transformation layers before rendering: `InlayMap` → `FoldMap` → `TabMap` → `WrapMap` → `BlockMap`. Each layer has its own coordinate space and snapshot, making the pipeline composable and debuggable. `DisplayMap` sits on top as the coordinator (adding highlights; `DisplayPoint` is a newtype of `BlockPoint`). `CreaseMap` is a metadata structure that tracks foldable regions (feeds into `FoldMap`), not a pipeline layer itself.

### 8. Settings Cascade

Settings merge hierarchically in this order (each layer overrides the previous):

```
default (compiled-in defaults)
  → extension (settings contributed by extensions)
  → global (global_settings — e.g. in SSH remote projects)
  → user (<platform-config-dir>/zed/settings.json, +release-channel/OS/profile variants)
    macOS:   ~/Library/Application Support/Zed/settings.json
    Linux:   $XDG_CONFIG_HOME/zed/settings.json  (~/.config/zed/settings.json)
    Windows: %APPDATA%\Zed\settings.json
  → server (pushed from the remote SSH machine when in a remote project)
  → project (.zed/settings.json, stacked per directory depth)
```

Language-specific and file/path-specific overrides are resolved at read time within the project layer via `SettingsLocation { worktree_id, path }`.

---

## Crate Map

Key crates and their roles:

| Crate | Role |
|-------|------|
| `gpui` | UI framework core |
| `gpui_platform` | Platform trait definitions |
| `gpui_macos`, `gpui_linux`, `gpui_windows` | Platform backends |
| `gpui_wgpu` | GPU rendering backend |
| `workspace` | Window layout, panes, panels |
| `project` | Data coordination (worktrees, buffers, LSP) |
| `worktree` | File system scanning and snapshots |
| `editor` | Text editor (UI + controls) |
| `multi_buffer` | Excerpt aggregation for editors |
| `language` | Buffer + syntax (Tree-sitter) |
| `language_core` | Core language types shared across crates |
| `language_model` | Abstraction layer for AI model providers |
| `language_models` | Concrete provider registration (registers all providers on startup) |
| `text` | Rope + CRDT core |
| `rope` | SumTree-based rope |
| `sum_tree` | B-tree with aggregate summaries; backbone of Rope and Worktree |
| `lsp` | LSP JSON-RPC client |
| `settings` | Settings store + traits |
| `extension`, `extension_host`, `extension_api` | Extension system |
| `collab` | Collaboration server |
| `client` | RPC client |
| `rpc` | RPC peer/client, WebSocket connection handling |
| `proto` | Protocol buffer definitions (cloud API) |
| `terminal` | Terminal emulation |
| `task` | Task templates and execution |
| `agent` | AI agent core: tools, templates, edit_agent, local LLM server |
| `agent_ui` | AI agent panel UI, inline assistant, conversation view |
| `agent_settings` | AI configuration |
| `anthropic`, `open_ai`, `google_ai`, `copilot_chat`, `ollama`, `bedrock`, `deepseek`, `codestral`, `open_router`, `cloud_llm_client`, ... | Model providers |
| `theme` | Theme loading and application |
| `fs` | Filesystem abstraction |
| `git` | Git integration |
| `dap` | Debug Adapter Protocol |

---

## Where to Start

**If you're new to the codebase:**
1. Start with [1_GPUI.md](1_GPUI.md) — understand the entity/context/render model
2. Read [2_Workspace-Pane.md](2_Workspace-Pane.md) — understand the UI layout
3. Read [4_Editor-Buffer.md](4_Editor-Buffer.md) — understand the editing engine

**If you're working on a specific area:**
- Language support → [6_LSP-Language-Server.md](6_LSP-Language-Server.md) + [7_Extensions.md](7_Extensions.md)
- Collaboration → [8_Collaboration-CRDT.md](8_Collaboration-CRDT.md)
- AI features → [10_Agent-AI.md](10_Agent-AI.md)
- Terminal/Tasks → [9_Terminal-Tasks.md](9_Terminal-Tasks.md)
- Configuration → [5_Settings-Registry.md](5_Settings-Registry.md)
