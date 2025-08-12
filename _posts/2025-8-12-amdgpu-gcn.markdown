---
layout: 
title:  "AMD GCN 架构深度分析"
date:   2025-8-12 7:00:16 +0800
categories: GPU 
---

<div class="custom-html">
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>交互式 GCN 渲染管线分析</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+SC:wght@300;400;500;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Noto Sans SC', sans-serif;
            background-color: #f8fafc; /* slate-50 */
        }
        .pipeline-stage {
            transition: all 0.3s ease-in-out;
            cursor: pointer;
        }
        .pipeline-stage.active {
            transform: translateY(-5px) scale(1.05);
            box-shadow: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
            background-color: #fbbf24; /* amber-400 */
            color: #1e293b; /* slate-800 */
        }
        .pipeline-stage.active .stage-icon {
             color: #1e293b;
        }
        .pipeline-connector {
            position: relative;
            flex-grow: 1;
            height: 2px;
            background-color: #94a3b8; /* slate-400 */
        }
        .pipeline-connector::after {
            content: '▶';
            position: absolute;
            right: -8px;
            top: 50%;
            transform: translateY(-50%);
            color: #94a3b8; /* slate-400 */
            font-size: 16px;
        }
        .content-section {
            display: none;
            animation: fadeIn 0.5s ease-in-out;
        }
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(20px); }
            to { opacity: 1; transform: translateY(0); }
        }
        .content-section.visible {
            display: block;
        }
        .cu-diagram .component {
            transition: background-color 0.2s ease;
            cursor: pointer; /* Add cursor pointer for click interaction */
        }
        .cu-diagram .component:hover {
            background-color: #d1d5db; /* gray-300 */
        }
        .wavefront-anim-bar {
            transition: width 1s ease-in-out;
        }
        .chart-container {
            position: relative;
            width: 100%;
            max-width: 600px;
            margin-left: auto;
            margin-right: auto;
            height: 350px;
            max-height: 400px;
        }
        @media (max-width: 768px) {
            .chart-container {
                height: 300px;
            }
        }
        .cu-box {
            width: 100%;
            padding-top: 100%; /* 1:1 Aspect Ratio */
            position: relative;
            background-color: #f1f5f9;
            border: 1px solid #e2e8f0;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 12px;
            font-weight: 500;
            color: #94a3b8;
            border-radius: 8px;
            text-align: center;
            transition: all 0.3s ease-in-out;
            overflow: hidden;
        }
        .cu-box .content {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            padding: 4px;
        }
        .cu-box.vertex {
            background-color: #93c5fd; /* blue-300 */
            color: #1e40af; /* blue-800 */
        }
        .cu-box.pixel {
            background-color: #fcd34d; /* amber-300 */
            color: #78350f; /* amber-900 */
        }
        .progress-bar {
            width: 80%;
            height: 8px;
            background-color: rgba(255, 255, 255, 0.5);
            border-radius: 4px;
            overflow: hidden;
            margin-top: 4px;
        }
        .progress {
            height: 100%;
            background-color: #3b82f6;
            transition: width 0.1s linear;
        }
        .cu-box.pixel .progress {
            background-color: #f59e0b;
        }
        
        /* Updated styles for the detailed wavefront animation */
        .wavefront-anim-container {
            display: flex;
            flex-direction: column;
            gap: 1rem;
            align-items: center;
            text-align: center;
        }
        .wavefront-instruction-box {
            padding: 0.5rem 1rem;
            background-color: #e2e8f0;
            border: 2px solid #94a3b8;
            border-radius: 8px;
            font-weight: 600;
            color: #1e293b;
            min-height: 40px;
            display: flex;
            align-items: center;
            justify-content: center;
            width: 100%;
            transition: all 0.3s ease;
        }
        .wavefront-instruction-box.running {
            background-color: #facc15; /* yellow-400 */
            color: #78350f;
            transform: scale(1.05);
        }
        .wavefront-cycles-container {
            display: flex;
            flex-direction: column;
            width: 100%;
            background-color: #e2e8f0;
            border: 1px solid #94a3b8;
            border-radius: 8px;
            overflow: hidden;
            position: relative;
        }
        .wavefront-cycle {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 0.5rem;
            border-bottom: 1px solid #94a3b8;
            transition: all 0.3s ease;
        }
        .wavefront-cycle:last-child {
            border-bottom: none;
        }
        .wavefront-cycle.active {
            background-color: #6ee7b7; /* emerald-300 */
        }
        .wavefront-legend {
            display: flex;
            justify-content: center;
            flex-wrap: wrap;
            gap: 1rem;
            margin-top: 1rem;
        }
        .wavefront-legend-item {
            display: flex;
            align-items: center;
            gap: 0.5rem;
            font-size: 0.875rem;
        }
        .wavefront-color-box {
            width: 16px;
            height: 16px;
            border-radius: 4px;
        }
        .wavefront-anim-step {
            flex: 1;
            padding: 0.25rem;
            border-radius: 4px;
            text-align: center;
            transition: all 0.2s ease;
        }
        .wavefront-anim-step.active {
            color: white;
            font-weight: 700;
            transform: scale(1.1);
        }
        .wavefront-progress-bar-container {
            width: 80px;
            height: 8px;
            background-color: #cbd5e1;
            border-radius: 4px;
            overflow: hidden;
        }
        .wavefront-progress-bar {
            height: 100%;
            background-color: #2563eb;
            transition: width 0.3s linear;
        }

        /* Updated styles for the memory path diagram */
        .memory-path-diagram {
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 1rem;
            padding: 1rem;
            border: 2px solid #94a3b8;
            border-radius: 0.5rem;
            background-color: #f1f5f9;
        }
        .memory-component {
            position: relative;
            padding: 0.5rem;
            background-color: #cbd5e1;
            border: 1px solid #94a3b8;
            border-radius: 0.25rem;
            font-weight: 600;
            color: #1e293b;
            text-align: center;
            width: 100%;
        }
        .memory-path-arrow {
            width: 2px;
            height: 1rem;
            background-color: #94a3b8;
            position: relative;
        }
        .memory-path-arrow::after {
            content: '▼';
            position: absolute;
            bottom: -8px;
            left: 50%;
            transform: translateX(-50%);
            color: #94a3b8;
        }
        .cu-cluster {
            display: flex;
            justify-content: space-around;
            gap: 0.5rem;
            width: 100%;
        }
        .cu-block {
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 0.5rem;
            flex: 1;
        }
        .l1-inst-cache {
             background-color: #fef9c3; /* yellow-100 */
             border: 1px solid #facc15; /* yellow-400 */
        }
        .l1-data-cache {
             background-color: #d1fae5; /* emerald-100 */
             border: 1px solid #34d399; /* emerald-400 */
        }
        .l2-cache-box {
            background-color: #818cf8; /* indigo-400 */
            border: 1px solid #3730a3; /* indigo-800 */
        }
        .mmu-tlb-box {
            background-color: #fde68a; /* amber-200 */
            border: 1px solid #b45309; /* amber-700 */
        }
        .connector-lines {
            position: relative;
            width: 100%;
            height: 2rem;
        }
        .vertical-line {
            position: absolute;
            width: 2px;
            background-color: #94a3b8;
            top: 0;
            height: 100%;
        }
        .horizontal-line {
            position: absolute;
            height: 2px;
            background-color: #94a3b8;
            top: 50%;
        }
        /* New and updated styles for the improved SPI animation */
        .spi-flow-container {
            display: flex;
            flex-direction: column;
            align-items: center;
            text-align: center;
            position: relative;
        }
        .spi-box {
            padding: 1rem;
            background-color: #e2e8f0;
            border: 2px solid #94a3b8;
            border-radius: 0.5rem;
            font-weight: 600;
            color: #1e293b;
            position: relative;
            z-index: 10;
            transition: all 0.5s ease-in-out;
        }
        .spi-box.processing {
            background-color: #facc15; /* yellow-400 */
            transform: scale(1.05);
        }
        .spi-box .progress-bar {
            width: 90%;
            height: 12px;
            background-color: #cbd5e1;
            border-radius: 6px;
            margin-top: 0.5rem;
        }
        .spi-box .progress {
            height: 100%;
            background-color: #1e40af; /* blue-800 */
            border-radius: 6px;
            transition: width 0.3s ease-out;
        }
        .spi-input-source {
            width: 100%;
            padding: 0.5rem;
            background-color: #dbeafe;
            border: 1px solid #93c5fd;
            border-radius: 0.25rem;
            margin-bottom: 1rem;
        }
        .spi-input-item {
            position: absolute;
            width: 8px;
            height: 8px;
            background-color: #3b82f6; /* blue-500 */
            border-radius: 50%;
            animation: moveInput 1s forwards;
        }
        @keyframes moveInput {
            from { top: -20px; opacity: 1; }
            to { top: 80px; opacity: 0; }
        }
        .spi-wavefronts-output {
            margin-top: 1rem;
            display: flex;
            flex-direction: column;
            align-items: center;
            min-height: 100px;
            gap: 0.5rem;
        }
        .wavefront-chip {
            padding: 0.5rem 1rem;
            background-color: #a5f3fc;
            border: 2px solid #22d3ee;
            border-radius: 0.5rem;
            font-size: 0.875rem;
            font-weight: 600;
            text-align: left;
            width: 100%;
            max-width: 250px;
            transform: translateY(20px) scale(0.9);
            opacity: 0;
            animation: createWavefront 0.5s ease-in-out forwards;
        }
        @keyframes createWavefront {
            to { transform: translateY(0) scale(1); opacity: 1; }
        }
        
        /* New styles for the detailed Scan Converter animation */
        .sc-flow-container {
            display: flex;
            flex-direction: column;
            align-items: center;
            text-align: center;
            position: relative;
        }
        .sc-box, .sc-output-box {
            padding: 1rem;
            background-color: #e2e8f0;
            border: 2px solid #94a3b8;
            border-radius: 0.5rem;
            font-weight: 600;
            color: #1e293b;
            position: relative;
            z-index: 10;
        }
        .sc-box.processing {
            background-color: #facc15;
            transform: scale(1.05);
            transition: all 0.5s ease-in-out;
        }
        .sc-output-box {
            min-height: 100px;
            margin-top: 1rem;
            display: flex;
            flex-wrap: wrap;
            gap: 0.25rem;
            justify-content: center;
            align-items: flex-start;
            padding: 0.5rem;
        }
        .sc-raster-grid {
            position: relative;
            width: 200px; /* Base width */
            height: 200px; /* Base height */
            background-color: #cbd5e1;
            border: 1px solid #94a3b8;
            display: grid;
            grid-template-columns: repeat(10, 1fr);
            grid-template-rows: repeat(10, 1fr);
            margin: 1rem auto;
            overflow: hidden;
        }
        .sc-grid-cell {
            position: relative;
            background-color: rgba(255, 255, 255, 0.1);
            border: 1px solid rgba(148, 163, 184, 0.5); /* Semi-transparent border */
        }
        .sc-triangle-poly {
            position: absolute;
            top: 0; left: 0;
            width: 100%; height: 100%;
            clip-path: polygon(10% 90%, 90% 90%, 50% 10%); /* Example triangle */
            background-color: rgba(59, 130, 246, 0.7); /* blue-500 with transparency */
            transition: all 0.5s ease-in-out;
        }
        .sc-quad {
            width: 40%;
            height: 40%;
            background-color: rgba(250, 204, 21, 0.5);
            position: absolute;
            top: var(--y);
            left: var(--x);
            transform: scale(1.1);
            border: 1px solid #ca8a04;
            transition: background-color 0.1s ease;
        }
        .sc-quad-highlight {
            background-color: #facc15;
        }
        .sc-scanner-marker {
            width: 16px;
            height: 16px;
            background-color: #dc2626;
            position: absolute;
            transform: translate(-50%, -50%);
            border-radius: 50%;
            transition: transform 0.1s linear;
        }
        .pixel-quad {
            width: 16px;
            height: 16px;
            background-color: #facc15;
            border: 1px solid #b45309;
            border-radius: 2px;
            transform: translateY(20px) scale(0.9);
            opacity: 0;
            animation: createQuad 0.5s ease-in-out forwards;
        }
        @keyframes createQuad {
            to { transform: translateY(0) scale(1); opacity: 1; }
        }
        /* New styles for the Texture Cache animation */
        .cache-container {
            display: flex;
            flex-direction: column;
            gap: 1rem;
            align-items: center;
        }
        .cache-line-container {
            display: flex;
            flex-direction: column;
            width: 100%;
            border: 2px solid #94a3b8;
            border-radius: 8px;
            background-color: #f1f5f9;
            padding: 0.5rem;
        }
        .cache-line {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 0.5rem;
            border-bottom: 1px solid #94a3b8;
            font-size: 0.875rem;
            background-color: #e2e8f0;
            transition: background-color 0.3s ease, transform 0.3s ease;
        }
        .cache-line:last-child {
            border-bottom: none;
        }
        .cache-line.hit {
            background-color: #6ee7b7; /* emerald-300 */
            transform: scale(1.02);
        }
        .cache-line.miss {
            background-color: #facc15; /* yellow-400 */
            transform: scale(1.02);
        }
        .cache-request-anim {
            position: relative;
            width: 20px;
            height: 20px;
            border-radius: 50%;
            background-color: #3b82f6; /* blue-500 */
            opacity: 0;
            transform: translateX(0);
            animation: none;
        }
        .cache-request-anim.hit-anim {
            animation: cacheHit 0.5s ease-out forwards;
        }
        .cache-request-anim.miss-anim {
            animation: cacheMiss 1.5s ease-in-out forwards;
        }
        @keyframes cacheHit {
            0% { opacity: 1; transform: translateX(-50px); }
            50% { opacity: 1; transform: translateX(0); }
            100% { opacity: 0; transform: translateX(0); }
        }
        @keyframes cacheMiss {
            0% { opacity: 1; transform: translateX(-50px); }
            30% { opacity: 1; transform: translateX(0); }
            70% { opacity: 1; transform: translateX(50px); }
            100% { opacity: 0; transform: translateX(50px); }
        }
        .memory-source {
            padding: 0.5rem 1rem;
            background-color: #dbeafe;
            border: 2px solid #93c5fd;
            border-radius: 8px;
            font-weight: 600;
            text-align: center;
            margin-top: 1rem;
        }
        .cache-status-item {
            font-size: 0.875rem;
        }
        .cache-status-item .hit {
            color: #10b981; /* emerald-600 */
        }
        .cache-status-item .miss {
            color: #eab308; /* yellow-600 */
        }
    </style>
</head>
<body class="bg-slate-50 text-slate-800">

    <div class="container mx-auto px-4 py-8 md:py-12">
        <!-- Chosen Palette: Warm Neutrals with Amber and Blue Accents -->
        <!-- Application Structure Plan: The SPA is designed as a single, vertically scrollable page. The top section features an interactive pipeline visualization acting as a main navigation bar. Clicking a stage scrolls the user to a detailed content block and highlights the stage, providing a linear exploration path. A second, below-the-fold section, "核心概念与优化," presents key architectural components and their functions as individual, interactive cards. This structure separates the high-level rendering process from the low-level hardware details. It is chosen to provide both a structured, guided tour (pipeline) and an unguided, focused exploration of individual concepts (cards), catering to different learning styles and allowing users to synthesize information logically. This design avoids deep hierarchies and keeps all information within a single, easy-to-navigate view. -->
        <!-- Visualization & Content Choices: The pipeline navigator is an HTML/CSS flexbox layout with click events to update text content blocks. The CU structure is an HTML grid with JS for hover effects and a detail panel update. The wavefront execution is a CSS/JS animation. The GPU task simulation uses a dynamic HTML grid and JS state management to show CU workload. The memory access path is a static HTML/CSS diagram with semantic labels. The performance chart uses Chart.js to dynamically visualize the relationship between VGPRs and occupancy. The new SPI card uses a simple HTML/CSS flow diagram and JS to animate the creation of wavefronts. The new Scan Converter card uses a simple animated flow. The new Texture Cache card visualizes cache hits and misses with a simple animation. All choices prioritize interactivity and dynamic data/text updates over static presentation to foster active learning, with a focus on avoiding complex libraries like Plotly and Mermaid to keep the application lightweight and responsive. NO SVG/Mermaid. -->
        <!-- CONFIRMATION: NO SVG graphics used. NO Mermaid JS used. -->
        <header class="text-center mb-12">
            <h1 class="text-4xl md:text-5xl font-bold text-slate-900">AMD GCN 架构深度分析</h1>
            <p class="mt-4 text-lg text-slate-600">一个关于三角形渲染管线与 CU 编排的交互式探索</p>
        </header>

        <main>
            <section id="pipeline-navigator" class="mb-12">
                <h2 class="text-2xl font-bold text-center mb-8 text-slate-700">交互式渲染管线</h2>
                <div class="flex flex-col md:flex-row items-center justify-center space-y-4 md:space-y-0 md:space-x-2">
                    <div class="pipeline-stage bg-white border border-slate-200 rounded-lg p-4 shadow-md text-center w-40" data-target="command">
                        <div class="stage-icon text-3xl mb-2 text-amber-500">📥</div>
                        <h3 class="font-semibold">命令提交</h3>
                    </div>
                    <div class="pipeline-connector w-px h-8 md:w-16 md:h-px"></div>
                    <div class="pipeline-stage bg-white border border-slate-200 rounded-lg p-4 shadow-md text-center w-40" data-target="vertex">
                        <div class="stage-icon text-3xl mb-2 text-amber-500">V</div>
                        <h3 class="font-semibold">顶点着色</h3>
                    </div>
                    <div class="pipeline-connector w-px h-8 md:w-16 md:h-px"></div>
                    <div class="pipeline-stage bg-white border border-slate-200 rounded-lg p-4 shadow-md text-center w-40" data-target="raster">
                        <div class="stage-icon text-3xl mb-2 text-amber-500">▦</div>
                        <h3 class="font-semibold">光栅化</h3>
                    </div>
                    <div class="pipeline-connector w-px h-8 md:w-16 md:h-px"></div>
                    <div class="pipeline-stage bg-white border border-slate-200 rounded-lg p-4 shadow-md text-center w-40" data-target="pixel">
                        <div class="stage-icon text-3xl mb-2 text-amber-500">P</div>
                        <h3 class="font-semibold">像素着色</h3>
                    </div>
                     <div class="pipeline-connector w-px h-8 md:w-16 md:h-px"></div>
                    <div class="pipeline-stage bg-white border border-slate-200 rounded-lg p-4 shadow-md text-center w-40" data-target="output">
                        <div class="stage-icon text-3xl mb-2 text-amber-500">📤</div>
                        <h3 class="font-semibold">输出合并</h3>
                    </div>
                </div>
                 <p class="text-center text-slate-500 mt-4">点击上方任一阶段以查看详细信息。</p>
            </section>
            
            <div id="details-container">
                <div id="welcome-message" class="text-center p-8 bg-white rounded-lg shadow-lg">
                    <h2 class="text-3xl font-bold text-slate-800 mb-4">欢迎！</h2>
                    <p class="text-slate-600 max-w-3xl mx-auto">
                        本应用将带您深入了解 AMD 的图形核心下一代 (GCN) 架构。GCN 标志着 GPU 设计的根本性转变，显著增强了图形渲染和通用计算能力。通过上方交互式管线，您可以逐步探索从命令提交到最终像素输出的整个三角形渲染过程。每个阶段都揭示了固定功能硬件与高度灵活的计算单元 (CU) 之间的复杂协同作用。
                    </p>
                </div>

                <div id="command-content" class="content-section bg-white rounded-lg shadow-lg p-6 md:p-8"></div>
                <div id="vertex-content" class="content-section bg-white rounded-lg shadow-lg p-6 md:p-8"></div>
                <div id="raster-content" class="content-section bg-white rounded-lg shadow-lg p-6 md:p-8"></div>
                <div id="pixel-content" class="content-section bg-white rounded-lg shadow-lg p-6 md:p-8"></div>
                <div id="output-content" class="content-section bg-white rounded-lg shadow-lg p-6 md:p-8"></div>
            </div>

            <section class="mt-16">
                 <h2 class="text-2xl font-bold text-center mb-8 text-slate-700">核心概念与优化</h2>
                 <div class="grid md:grid-cols-2 lg:grid-cols-3 gap-8">
                    <!-- New Detailed Scan Converter Card -->
                    <div class="bg-white rounded-lg shadow-lg p-6 relative">
                        <h3 class="text-xl font-bold mb-4">光栅化与扫描转换器</h3>
                        <p class="text-slate-600 mb-4">扫描转换器（SC）是固定功能硬件，负责将三角形等几何图元转换为像素。它将像素分组为 2x2 的“四边形”并发送到 SPI，以实现高效的纹理采样。以下动画展示了扫描转换器的工作流程。</p>
                        <div class="sc-flow-container text-sm">
                            <div class="sc-box relative w-full p-4" id="sc-box">
                                <span class="font-bold">扫描转换器 (SC)</span>
                                <p class="text-xs mt-1 text-slate-600">将几何转换为像素四边形</p>
                                <div class="sc-raster-grid relative" id="sc-raster-grid">
                                    <div class="sc-triangle-poly" id="sc-triangle-poly"></div>
                                </div>
                                <div id="sc-status" class="text-xs text-slate-500 mt-2">点击“开始”按钮</div>
                            </div>
                            <div class="w-2 h-4 bg-slate-400 my-2"></div>
                            <div class="sc-output-box relative" id="sc-output">
                                <p class="text-xs text-slate-500 absolute top-2 left-1/2 -translate-x-1/2">输出: 像素四边形</p>
                            </div>
                            <p class="text-xs mt-2 text-slate-600">已生成四边形: <span id="sc-quad-count" class="font-bold">0</span></p>
                        </div>
                        <div class="flex space-x-2 mt-4">
                            <button id="sc-simulate-btn" class="flex-1 bg-green-500 text-white font-bold py-2 px-4 rounded hover:bg-green-600 transition-colors">开始光栅化</button>
                            <button id="sc-reset-btn" class="flex-1 bg-slate-800 text-white font-bold py-2 px-4 rounded hover:bg-slate-700 transition-colors">重置</button>
                        </div>
                    </div>
                    <!-- End of New Scan Converter Card -->

                    <div class="bg-white rounded-lg shadow-lg p-6 relative">
                        <h3 class="text-xl font-bold mb-4">着色器处理器输入 (SPI)</h3>
                        <p class="text-slate-600 mb-4">SPI 是连接管线固定功能硬件与可编程计算单元的关键。其核心任务是收集几何和像素数据，并将它们组织成可由 CU 并行执行的“波前”。</p>
                        <div class="spi-flow-container text-sm">
                            <div class="spi-input-source" id="spi-input">
                                <span>输入: 顶点 / 像素四边形</span>
                                <span class="font-bold text-xs block mt-1" id="spi-status-text">已收集: <span id="spi-count">0</span> / 64</span>
                            </div>
                            <div class="spi-box mt-4 relative" id="spi-box">
                                <span class="font-bold">着色器处理器输入 (SPI)</span>
                                <p class="text-xs mt-1 text-slate-600">累积数据并形成波前</p>
                                <div class="progress-bar mt-2 mx-auto"><div class="progress" id="spi-progress" style="width: 0%;"></div></div>
                            </div>
                            <div class="spi-wavefronts-output" id="spi-wavefronts-output">
                                 <!-- Wavefront chips will be dynamically added here -->
                            </div>
                        </div>
                        <div class="flex space-x-2 mt-4">
                            <button id="spi-simulate-btn" class="flex-1 bg-green-500 text-white font-bold py-2 px-4 rounded hover:bg-green-600 transition-colors">开始模拟</button>
                            <button id="spi-reset-btn" class="flex-1 bg-slate-800 text-white font-bold py-2 px-4 rounded hover:bg-slate-700 transition-colors">重置</button>
                        </div>
                    </div>
                    <div class="bg-white rounded-lg shadow-lg p-6">
                        <h3 class="text-xl font-bold mb-4">计算单元 (CU) 结构</h3>
                        <p class="text-slate-600 mb-4">CU 是 GCN 的核心处理单元。每个 CU 都包含向量和标量处理器、共享内存和缓存，旨在高效执行大量并行任务。点击下方组件以了解其功能。</p>
                        <div class="cu-diagram border-2 border-slate-400 rounded p-2 text-center bg-slate-200">
                            <div class="font-bold text-slate-700">计算单元 (CU)</div>
                            <div class="grid grid-cols-2 gap-2 mt-2">
                                <div class="component bg-slate-100 p-2 rounded border border-slate-300" data-info="每个 CU 由 4 个 16 通道宽度的 SIMD 向量单元组成。它们是执行大部分向量指令的核心，例如在像素着色器中计算颜色。">SIMD-VU 0</div>
                                <div class="component bg-slate-100 p-2 rounded border border-slate-300" data-info="每个 CU 由 4 个 16 通道宽度的 SIMD 向量单元组成。它们是执行大部分向量指令的核心，例如在像素着色器中计算颜色。">SIMD-VU 1</div>
                                <div class="component bg-slate-100 p-2 rounded border border-slate-300" data-info="每个 CU 由 4 个 16 通道宽度的 SIMD 向量单元组成。它们是执行大部分向量指令的核心，例如在像素着色器中计算颜色。">SIMD-VU 2</div>
                                <div class="component bg-slate-100 p-2 rounded border border-slate-300" data-info="每个 CU 由 4 个 16 通道宽度的 SIMD 向量单元组成。它们是执行大部分向量指令的核心，例如在像素着色器中计算颜色。">SIMD-VU 3</div>
                            </div>
                            <div class="grid grid-cols-2 gap-2 mt-2">
                                <div class="component bg-slate-100 p-2 rounded border border-slate-300" data-info="标量单元负责处理所有非向量指令，包括控制流（分支、循环）和标量数据计算。">标量单元</div>
                                <div class="component bg-slate-100 p-2 rounded border border-slate-300" data-info="局部数据共享（LDS）是 64 KiB 的高速片上内存，位于 CU 内部。它允许同一工作组中的线程共享数据，是高效计算的重要组成部分。">局部数据共享 (LDS)</div>
                            </div>
                        </div>
                        <div id="cu-details" class="mt-4 p-4 text-sm bg-slate-100 border border-slate-300 rounded-md text-slate-700 hidden">
                             点击上方的组件以查看详细信息。
                        </div>
                    </div>

                    <!-- Updated Wavefront Animation Card with slider -->
                    <div class="bg-white rounded-lg shadow-lg p-6">
                        <h3 class="text-xl font-bold mb-4">波前 (Wavefront) 交错执行</h3>
                        <p class="text-slate-600 mb-4">CU 通过交错执行多个波前指令来隐藏延迟。每个波前指令需要被调度四次才能完成。此动画模拟了多波前如何共享 CU 资源。</p>
                        <div class="wavefront-anim-container">
                             <div class="mt-2 px-4 w-full">
                                <label for="wavefront-slider" class="block font-medium text-center text-slate-700">活跃波前数量: <span id="wavefront-count" class="font-bold text-amber-600">3</span></label>
                                <input id="wavefront-slider" type="range" min="1" max="8" value="3" step="1" class="w-full h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer">
                            </div>
                            <div class="wavefront-legend" id="wavefront-legend">
                                <!-- Legend items will be dynamically generated here -->
                            </div>
                            <div class="wavefront-instruction-box mt-4" id="wavefront-instruction">点击播放/重置动画</div>
                            <div class="wavefront-cycles-container" id="wavefront-cycles-container">
                                <div class="flex justify-between p-2 border-b border-gray-400">
                                    <span class="font-bold">时钟周期</span>
                                    <span class="font-bold text-center w-1/3">执行中</span>
                                    <span class="font-bold text-right w-1/3">指令进度</span>
                                </div>
                            </div>
                        </div>
                        <button id="play-anim-btn" class="mt-4 w-full bg-slate-800 text-white font-bold py-2 px-4 rounded hover:bg-slate-700 transition-colors">播放/重置动画</button>
                    </div>
                    <!-- End of Updated Wavefront Animation Card -->
                    
                    <div class="bg-white rounded-lg shadow-lg p-6 md:col-span-2 lg:col-span-1">
                        <h3 class="text-xl font-bold mb-4">GPU 任务并发执行模拟</h3>
                        <p class="text-slate-600 mb-4">此模拟展示了 2560 个顶点在 16 个 CU 上的渲染过程。当有足够的四边形形成波前时，像素着色器任务会与顶点着色器任务并发执行。</p>
                        <div class="flex items-center justify-between text-sm mb-4 bg-slate-100 rounded-lg p-3">
                            <div class="text-center">
                                <p class="font-semibold text-blue-600">顶点任务</p>
                                <span id="vertex-tasks-left" class="font-bold text-lg">40</span> / 40
                            </div>
                            <div class="text-center">
                                <p class="font-semibold text-amber-600">像素任务</p>
                                <span id="pixel-tasks-left" class="font-bold text-lg">80</span> / 80
                            </div>
                            <div class="text-center">
                                <p class="font-semibold text-slate-600">已完成</p>
                                <span id="tasks-completed" class="font-bold text-lg">0</span> / 120
                            </div>
                        </div>
                        <div id="cu-grid" class="grid grid-cols-4 gap-2 p-4 bg-slate-200 rounded-lg">
                             <!-- CU boxes will be dynamically generated here -->
                        </div>
                        <div class="flex space-x-2 mt-4">
                            <button id="start-simulation-btn" class="flex-1 bg-green-500 text-white font-bold py-2 px-4 rounded hover:bg-green-600 transition-colors">开始模拟</button>
                            <button id="reset-simulation-btn" class="flex-1 bg-slate-800 text-white font-bold py-2 px-4 rounded hover:bg-slate-700 transition-colors">重置模拟</button>
                        </div>
                    </div>

                    <div class="bg-white rounded-lg shadow-lg p-6">
                        <h3 class="text-xl font-bold mb-4">内存访问路径：缓存与 TLB</h3>
                        <p class="text-slate-600 mb-4">GPU 采用分层的缓存结构，以减少对显存的访问。CU 在读取指令和数据时，会先访问 L1 缓存，然后是 L2，最后才访问显存。这种设计是实现高性能的关键。</p>
                         <div class="memory-path-diagram">
                            <div class="memory-component l1-inst-cache w-full max-w-lg">共享 L1 指令缓存 (SQC)</div>
                            <div class="connector-lines">
                                <div class="vertical-line" style="left: 50%; height: 1.5rem;"></div>
                                <div class="horizontal-line" style="left: 12.5%; width: 75%; top: 1.5rem;"></div>
                                <div class="vertical-line" style="left: 12.5%; top: 1.5rem; height: 1.5rem;"></div>
                                <div class="vertical-line" style="left: 37.5%; top: 1.5rem; height: 1.5rem;"></div>
                                <div class="vertical-line" style="left: 62.5%; top: 1.5rem; height: 1.5rem;"></div>
                                <div class="vertical-line" style="left: 87.5%; top: 1.5rem; height: 1.5rem;"></div>
                            </div>
                            <div class="cu-cluster">
                                <div class="memory-component bg-blue-300">CU 1</div>
                                <div class="memory-component bg-blue-300">CU 2</div>
                                <div class="memory-component bg-blue-300">CU 3</div>
                                <div class="memory-component bg-blue-300">CU 4</div>
                            </div>
                            <div class="cu-cluster">
                                <div class="memory-path-arrow"></div>
                                <div class="memory-path-arrow"></div>
                                <div class="memory-path-arrow"></div>
                                <div class="memory-path-arrow"></div>
                            </div>
                            <div class="cu-cluster">
                                <div class="memory-component l1-data-cache">L1 数据缓存</div>
                                <div class="memory-component l1-data-cache">L1 数据缓存</div>
                                <div class="memory-component l1-data-cache">L1 数据缓存</div>
                                <div class="memory-component l1-data-cache">L1 数据缓存</div>
                            </div>
                            <div class="connector-lines">
                                <div class="vertical-line" style="left: 12.5%; height: 1.5rem;"></div>
                                <div class="vertical-line" style="left: 37.5%; height: 1.5rem;"></div>
                                <div class="vertical-line" style="left: 62.5%; height: 1.5rem;"></div>
                                <div class="vertical-line" style="left: 87.5%; height: 1.5rem;"></div>
                                <div class="horizontal-line" style="left: 12.5%; width: 75%; top: 1.5rem;"></div>
                                <div class="vertical-line" style="left: 50%; top: 1.5rem; height: 1.5rem;"></div>
                            </div>
                            <div class="memory-component l2-cache-box w-full max-w-lg">共享 L2 缓存</div>
                            <div class="memory-path-arrow"></div>
                            <div class="memory-component mmu-tlb-box w-full max-w-lg">MMU / TLB</div>
                            <div class="memory-path-arrow"></div>
                            <div class="memory-component bg-gray-400 w-full max-w-lg">HBM/GDDR 显存</div>
                        </div>
                    </div>


                    <div class="bg-white rounded-lg shadow-lg p-6">
                        <h3 class="text-xl font-bold mb-4">异步计算</h3>
                        <p class="text-slate-600 mb-4">当图形管线因固定功能单元（如 ROP）而出现瓶颈时，异步计算允许 GPU 利用空闲的 CU 周期执行计算任务，从而提高整体利用率。</p>
                        <div class="space-y-2">
                            <p class="font-semibold">图形管线:</p>
                            <div class="w-full bg-slate-200 rounded h-6 flex items-center border border-slate-300">
                                <div class="bg-blue-400 h-full" style="width: 30%"></div>
                                <div class="bg-slate-200 h-full" style="width: 20%"></div>
                                <div class="bg-blue-400 h-full" style="width: 50%"></div>
                            </div>
                             <p class="font-semibold">计算任务:</p>
                            <div class="w-full bg-slate-200 rounded h-6 flex items-center border border-slate-300">
                                 <div class="bg-slate-200 h-full" style="width: 30%"></div>
                                 <div id="async-fill" class="bg-green-400 h-full rounded-sm" style="width: 0%; transition: width 0.5s ease-in-out;"></div>
                            </div>
                        </div>
                         <button id="toggle-async-btn" class="mt-4 w-full bg-slate-800 text-white font-bold py-2 px-4 rounded hover:bg-slate-700 transition-colors">启用异步计算</button>
                    </div>


                    <div class="bg-white rounded-lg shadow-lg p-6 md:col-span-2 lg:col-span-3">
                        <h3 class="text-xl font-bold mb-4 text-center">性能优化：占用率与寄存器压力</h3>
                        <p class="text-slate-600 mb-4 text-center max-w-3xl mx-auto">CU 占用率是衡量其繁忙程度的关键指标。着色器使用的向量通用寄存器 (VGPR) 数量会直接影响可并发运行的波前数量。高 VGPR 压力会减少活动波前，从而降低占用率和隐藏内存延迟的能力。使用下面的滑块来观察这种关系。</p>
                        <div class="chart-container">
                            <canvas id="occupancyChart"></canvas>
                        </div>
                        <div class="mt-4 px-4">
                            <label for="vgpr-slider" class="block font-medium text-center text-slate-700">每个线程的 VGPR 数量: <span id="vgpr-value" class="font-bold text-amber-600">32</span></label>
                            <input id="vgpr-slider" type="range" min="16" max="256" value="32" step="1" class="w-full h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer">
                        </div>
                    </div>
                 </div>
            </section>
        </main>
        
        <footer class="text-center mt-16 pt-8 border-t border-slate-200">
            <p class="text-slate-500">基于 AMD GCN 架构分析报告生成的交互式应用。</p>
        </footer>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', function() {
            // --- Content Data & Pipeline Logic ---
            const contentData = {
                command: {
                    title: 'A. 命令提交和初始调度',
                    content: `
                        <p class="text-slate-600 mb-4">渲染过程始于主机 CPU，它将高级 API 调用（如 DirectX 或 Vulkan）转换为低级 GCN 命令。这些命令被写入内存中的命令缓冲区，然后提交给 GPU 的命令处理器。</p>
                        <ul class="list-disc list-inside space-y-2 text-slate-600">
                            <li><strong>图形命令处理器 (GCP):</strong> 专门处理图形队列，负责编排传统的渲染管线。</li>
                            <li><strong>异步计算引擎 (ACE)::</strong> 并行处理计算队列，实现了图形与计算任务的重叠执行。</li>
                            <li><strong>API 调用:</strong> 诸如 <code>vkCmdBindPipeline</code> 和 <code>vkCmdDrawIndexed</code> 等调用，会告知 GPU 要访问哪些状态和数据以进行后续的绘制，为渲染任务做好准备。</li>
                        </ul>
                    `
                },
                vertex: {
                    title: 'B. 顶点着色',
                    content: `
                        <p class="text-slate-600 mb-4">此阶段处理单个顶点，转换其位置并计算后续阶段所需的其他属性（如颜色、纹理坐标）。</p>
                        <ul class="list-disc list-inside space-y-2 text-slate-600">
                            <li><strong>输入装配器 (IA):</strong> 从内存中获取顶点数据，并将其组装成图元（如三角形）。</li>
                            <li><strong>着色器处理器输入 (SPI):</strong> 累积顶点，直到形成一个包含 64 个顶点的波前。</li>
                            <li><strong>CU 执行:</strong> SPI 将顶点着色器波前调度到可用的计算单元 (CU) 上。CU 在 64 个顶点的波前上并行执行顶点着色器程序，输出转换后的顶点。</li>
                        </ul>
                    `
                },
                raster: {
                    title: 'D. 光栅化',
                    content: `
                        <p class="text-slate-600 mb-4">光栅化是连接几何域（三角形）与像素域的关键桥梁。此阶段由固定功能的硬件处理，以实现最高效率。</p>
                        <ul class="list-disc list-inside space-y-2 text-slate-600">
                            <li><strong>扫描转换器 (SC):</strong> 接收来自几何阶段的三角形，并确定每个三角形所覆盖的屏幕像素。</li>
                            <li><strong>生成“四边形”:</strong> 为提高效率，扫描转换器通常以 2x2 的像素块（称为“四边形”）为单位进行操作。这对于计算纹理采样所需的导数至关重要。</li>
                            <li><strong>转发至 SPI:</strong> 覆盖的像素/片段（以四边形形式）连同其插值属性被转发到着色器处理器输入 (SPI)，为像素着色阶段做准备。</li>
                        </ul>
                    `
                },
                pixel: {
                    title: 'E. 像素着色',
                    content: `
                        <p class="text-slate-600 mb-4">这是渲染管线中计算量最大的阶段之一，负责计算每个像素的最终颜色。</p>
                        <ul class="list-disc list-inside space-y-2 text-slate-600">
                            <li><strong>SPI 累积:</strong> SPI 等待，直到有足够的像素四边形形成一个像素着色器波前（每波 64 个像素/片段）。</li>
                            <li><strong>CU 执行:</strong> CU 并行执行像素着色器程序，结合纹理数据、插值属性和其他信息来生成逐像素输出。</li>
                            <li><strong>线程屏蔽:</strong> 如果一个三角形未完全覆盖一个四边形，则未被覆盖的像素对应的线程将被屏蔽，以避免不必要的计算。</li>
                            <li><strong>着色器导出 (SX)::</strong> 完成的像素颜色和深度值通过着色器导出单元导出，送往下一阶段。</li>
                        </ul>
                    `
                },
                output: {
                    title: 'F. 输出合并',
                    content: `
                        <p class="text-slate-600 mb-4">这是管线的最后阶段，处理后的像素被写入最终的图像（帧缓冲区）。此阶段由专用的固定功能硬件——渲染输出单元 (ROP) 处理。</p>
                        <ul class="list-disc list-inside space-y-2 text-slate-600">
                            <li><strong>深度/模板测试:</strong> ROP 执行测试以确定像素是否可见（即没有被其他物体遮挡）。</li>
                            <li><strong>混合:</strong> 将新计算出的像素颜色与帧缓冲区中已有的颜色进行组合（例如，用于实现半透明效果）。</li>
                            <li><strong>写入帧缓冲区:</strong> 最终的像素颜色被写入 GPU 内存中的帧缓冲区，完成渲染过程。</li>
                        </ul>
                    `
                }
            };
            const pipelineStages = document.querySelectorAll('.pipeline-stage');
            const detailsContainer = document.getElementById('details-container');
            const welcomeMessage = document.getElementById('welcome-message');

            pipelineStages.forEach(stage => {
                stage.addEventListener('click', () => {
                    const targetId = stage.dataset.target;
                    pipelineStages.forEach(s => s.classList.remove('active'));
                    stage.classList.add('active');
                    welcomeMessage.style.display = 'none';
                    document.querySelectorAll('.content-section').forEach(section => {
                        section.classList.remove('visible');
                    });
                    const targetContentEl = document.getElementById(`${targetId}-content`);
                    const data = contentData[targetId];
                    targetContentEl.innerHTML = `<h2 class="text-3xl font-bold mb-6 text-slate-800">${data.title}</h2>${data.content}`;
                    targetContentEl.classList.add('visible');
                    targetContentEl.scrollIntoView({ behavior: 'smooth', block: 'center' });
                });
            });

            // --- Updated Wavefront Animation Logic for Interleaving ---
            const playAnimBtn = document.getElementById('play-anim-btn');
            const wavefrontInstruction = document.getElementById('wavefront-instruction');
            const cyclesContainer = document.getElementById('wavefront-cycles-container');
            const wavefrontSlider = document.getElementById('wavefront-slider');
            const wavefrontCountEl = document.getElementById('wavefront-count');
            const wavefrontLegend = document.getElementById('wavefront-legend');
            const ANIMATION_DELAY = 700; // ms
            const TOTAL_CYCLES_PER_INSTRUCTION = 4;
            const WAVEFRONT_COLORS = ['#3b82f6', '#10b981', '#8b5cf6', '#ef4444', '#f59e0b', '#06b6d4', '#475569', '#a21caf'];
            const WAVEFRONT_INSTRUCTIONS = ['V_ADD_F32', 'V_MUL_F32', 'V_MAX_F32', 'V_MIN_F32', 'V_LSHL_B32', 'V_ASHR_I32', 'V_CMP_GT_F32', 'V_SUB_I32'];

            function updateWavefrontLegend(numWavefronts) {
                wavefrontLegend.innerHTML = '';
                for (let i = 0; i < numWavefronts; i++) {
                    const legendItem = document.createElement('div');
                    legendItem.classList.add('wavefront-legend-item');
                    legendItem.innerHTML = `<div class="wavefront-color-box" style="background-color: ${WAVEFRONT_COLORS[i]};"></div>波前 ${String.fromCharCode(65 + i)}`;
                    wavefrontLegend.appendChild(legendItem);
                }
            }
            
            wavefrontSlider.addEventListener('input', (e) => {
                wavefrontCountEl.textContent = e.target.value;
                resetWavefrontAnimation();
                updateWavefrontLegend(e.target.value);
            });

            function resetWavefrontAnimation() {
                // Reset the instruction box color and text
                wavefrontInstruction.textContent = '点击播放/重置动画';
                wavefrontInstruction.style.backgroundColor = '';
                wavefrontInstruction.style.color = '';
                wavefrontInstruction.classList.remove('running');
                
                playAnimBtn.disabled = false;
                playAnimBtn.textContent = '播放/重置动画';
                if(animationInterval) {
                     clearInterval(animationInterval);
                     animationInterval = null;
                }
                cyclesContainer.innerHTML = `
                    <div class="flex justify-between p-2 border-b border-gray-400">
                        <span class="font-bold">时钟周期</span>
                        <span class="font-bold text-center w-1/3">执行中</span>
                        <span class="font-bold text-right w-1/3">指令进度</span>
                    </div>`;
            }

            let animationInterval = null;

            function playWavefrontAnimation() {
                if(animationInterval) {
                    clearInterval(animationInterval);
                    animationInterval = null;
                    resetWavefrontAnimation();
                    return;
                }

                resetWavefrontAnimation();
                playAnimBtn.disabled = true;

                const numWavefronts = parseInt(wavefrontSlider.value);
                const wavefronts = [];
                for (let i = 0; i < numWavefronts; i++) {
                    wavefronts.push({
                        id: String.fromCharCode(65 + i),
                        color: WAVEFRONT_COLORS[i],
                        completedCycles: 0,
                        totalCycles: TOTAL_CYCLES_PER_INSTRUCTION,
                        instruction: WAVEFRONT_INSTRUCTIONS[i]
                    });
                }
                
                let currentCycleNumber = 0;
                let activeWavefrontIndex = 0;

                const animateStep = () => {
                    const allDone = wavefronts.every(wf => wf.completedCycles >= wf.totalCycles);
                    if (allDone) {
                        wavefrontInstruction.textContent = '所有波前指令执行完成，CU 空闲';
                        wavefrontInstruction.style.backgroundColor = ''; // Reset color
                        wavefrontInstruction.style.color = '';
                        wavefrontInstruction.classList.remove('running');
                        playAnimBtn.disabled = false;
                        playAnimBtn.textContent = '播放/重置动画';
                        clearInterval(animationInterval);
                        animationInterval = null;
                        return;
                    }

                    currentCycleNumber++;
                    const currentWavefront = wavefronts[activeWavefrontIndex];
                    
                    // Only process if the wavefront still has cycles to complete
                    if (currentWavefront.completedCycles < currentWavefront.totalCycles) {
                        currentWavefront.completedCycles++;

                        const cycleRow = document.createElement('div');
                        cycleRow.classList.add('flex', 'justify-between', 'items-center', 'p-2', 'border-b', 'border-gray-200');
                        cycleRow.innerHTML = `
                            <span>时钟周期 ${currentCycleNumber}</span>
                            <span class="font-bold px-2 py-1 rounded-md text-white w-1/3 text-center" style="background-color: ${currentWavefront.color};">波前 ${currentWavefront.id}</span>
                            <span class="font-bold text-right w-1/3">${currentWavefront.completedCycles}/${currentWavefront.totalCycles}</span>
                        `;
                        cyclesContainer.appendChild(cycleRow);
                        cyclesContainer.scrollTop = cyclesContainer.scrollHeight;

                        wavefrontInstruction.textContent = `执行中：波前 ${currentWavefront.id} (${currentWavefront.instruction}), 进度：${currentWavefront.completedCycles}/${currentWavefront.totalCycles}`;
                        wavefrontInstruction.style.backgroundColor = currentWavefront.color;
                        wavefrontInstruction.style.color = 'white'; // Use white text for better contrast
                        wavefrontInstruction.classList.add('running');

                    } else {
                        // If this wavefront is already done, skip to the next one
                        wavefrontInstruction.textContent = `时钟周期 ${currentCycleNumber}: 波前 ${currentWavefront.id} 已完成，CU 切换到下一个。`;
                        wavefrontInstruction.style.backgroundColor = '#e2e8f0'; // Default gray color
                        wavefrontInstruction.style.color = '#1e293b'; // Default text color
                        wavefrontInstruction.classList.remove('running');
                    }

                    // Find the next active wavefront that is not yet completed
                    let nextIndex = (activeWavefrontIndex + 1) % wavefronts.length;
                    let attempts = 0;
                    while (wavefronts[nextIndex].completedCycles >= wavefronts[nextIndex].totalCycles && attempts < wavefronts.length) {
                         nextIndex = (nextIndex + 1) % wavefronts.length;
                         attempts++;
                    }
                    activeWavefrontIndex = nextIndex;
                };
                
                wavefrontInstruction.textContent = '开始调度波前指令...';
                animationInterval = setInterval(animateStep, ANIMATION_DELAY);
            }

            playAnimBtn.addEventListener('click', playWavefrontAnimation);
            updateWavefrontLegend(wavefrontSlider.value);
                       
            // --- Improved SPI Simulation Logic ---
            const spiSimulateBtn = document.getElementById('spi-simulate-btn');
            const spiResetBtn = document.getElementById('spi-reset-btn');
            const spiBoxEl = document.getElementById('spi-box');
            const spiCountEl = document.getElementById('spi-count');
            const spiProgressEl = document.getElementById('spi-progress');
            const spiWavefrontsOutputEl = document.getElementById('spi-wavefronts-output');
            const spiInputEl = document.getElementById('spi-input');

            const WAVEFRONT_SIZE = 64;
            let currentSpiCount = 0;
            let wavefrontCounter = 0;
            let simulationInterval = null;

            function animateInputItem() {
                const item = document.createElement('div');
                item.classList.add('spi-input-item');
                spiInputEl.appendChild(item);
                
                const startX = Math.random() * (spiInputEl.offsetWidth - 8);
                item.style.left = `${startX}px`;
                
                // Use a keyframe animation and then remove the element
                item.addEventListener('animationend', () => {
                    item.remove();
                });
            }

            function createWavefront() {
                wavefrontCounter++;
                const newWavefront = document.createElement('div');
                newWavefront.classList.add('wavefront-chip');
                newWavefront.innerHTML = `
                    <span>波前 ${wavefrontCounter}</span>
                    <span class="param">PC: 0x${(Math.random() * 0xFFFFFF).toString(16).slice(0, 6)}...</span>
                    <span class="param">VGPRs: 32</span>
                    <span class="param">SGPRs: 16</span>
                    <span class="param">Thread IDs: 0-${WAVEFRONT_SIZE-1}</span>
                `;
                spiWavefrontsOutputEl.appendChild(newWavefront);
            }

            function simulationStep() {
                if (currentSpiCount < WAVEFRONT_SIZE * 3) { // Simulate 3 wavefronts
                    currentSpiCount++;
                    spiCountEl.textContent = currentSpiCount % WAVEFRONT_SIZE;
                    spiProgressEl.style.width = `${(currentSpiCount % WAVEFRONT_SIZE) / WAVEFRONT_SIZE * 100}%`;
                    animateInputItem();

                    if (currentSpiCount > 0 && currentSpiCount % WAVEFRONT_SIZE === 0) {
                        spiBoxEl.classList.add('processing');
                        setTimeout(() => {
                             spiProgressEl.style.width = `0%`;
                             spiCountEl.textContent = '0';
                             createWavefront();
                             spiBoxEl.classList.remove('processing');
                        }, 500); // Wait a bit for processing
                    }
                } else {
                    clearInterval(simulationInterval);
                    simulationInterval = null;
                    spiSimulateBtn.textContent = '模拟完成！';
                    spiSimulateBtn.disabled = true;
                }
            }

            function startSpiSimulation() {
                if (!simulationInterval) {
                    currentSpiCount = 0;
                    wavefrontCounter = 0;
                    spiWavefrontsOutputEl.innerHTML = '';
                    spiProgressEl.style.width = '0%';
                    spiCountEl.textContent = '0';
                    spiSimulateBtn.textContent = '暂停模拟';
                    spiSimulateBtn.classList.remove('bg-green-500', 'hover:bg-green-600');
                    spiSimulateBtn.classList.add('bg-amber-500', 'hover:bg-amber-600');
                    spiResetBtn.disabled = false;
                    simulationInterval = setInterval(simulationStep, 50); // Fast simulation
                } else {
                    clearInterval(simulationInterval);
                    simulationInterval = null;
                    spiSimulateBtn.textContent = '继续模拟';
                    spiSimulateBtn.classList.remove('bg-amber-500', 'hover:bg-amber-600');
                    spiSimulateBtn.classList.add('bg-green-500', 'hover:bg-green-600');
                }
            }
            
            function resetSpiSimulation() {
                clearInterval(simulationInterval);
                simulationInterval = null;
                currentSpiCount = 0;
                wavefrontCounter = 0;
                spiProgressEl.style.width = '0%';
                spiCountEl.textContent = '0';
                spiWavefrontsOutputEl.innerHTML = '';
                spiSimulateBtn.textContent = '开始模拟';
                spiSimulateBtn.classList.remove('bg-amber-500', 'hover:bg-amber-600');
                spiSimulateBtn.classList.add('bg-green-500', 'hover:bg-green-600');
                spiSimulateBtn.disabled = false;
                spiResetBtn.disabled = true;
            }

            spiSimulateBtn.addEventListener('click', startSpiSimulation);
            spiResetBtn.addEventListener('click', resetSpiSimulation);


            // --- New Detailed Scan Converter Simulation Logic ---
            const scSimulateBtn = document.getElementById('sc-simulate-btn');
            const scResetBtn = document.getElementById('sc-reset-btn');
            const scBoxEl = document.getElementById('sc-box');
            const scRasterGridEl = document.getElementById('sc-raster-grid');
            const scOutputEl = document.getElementById('sc-output');
            const scQuadCountEl = document.getElementById('sc-quad-count');
            const scStatusEl = document.getElementById('sc-status');
            
            const GRID_SIZE = 10;
            const CELL_SIZE = 20; // in px
            const TRIANGLE_VERTICES = [
                {x: 2, y: 8}, {x: 8, y: 8}, {x: 5, y: 2} // Grid coordinates
            ];

            let scSimulationInterval = null;
            let currentQuadCount = 0;
            let scanPosition = {x: 0, y: 0};
            
            function createRasterGrid() {
                 scRasterGridEl.style.width = `${GRID_SIZE * CELL_SIZE}px`;
                 scRasterGridEl.style.height = `${GRID_SIZE * CELL_SIZE}px`;
                 scRasterGridEl.style.gridTemplateColumns = `repeat(${GRID_SIZE}, 1fr)`;
                 scRasterGridEl.style.gridTemplateRows = `repeat(${GRID_SIZE}, 1fr)`;
                 scRasterGridEl.innerHTML = '';
                 for (let i = 0; i < GRID_SIZE * GRID_SIZE; i++) {
                     const cell = document.createElement('div');
                     cell.classList.add('sc-grid-cell');
                     scRasterGridEl.appendChild(cell);
                 }
                 scRasterGridEl.innerHTML += `<div class="sc-triangle-poly"></div>`;
            }

            function isInsideTriangle(x, y) {
                const p = {x, y};
                const v = TRIANGLE_VERTICES;
                
                // Using barycentric coordinates or cross product method
                // Check if point p is on the same side of each edge
                function sign(p1, p2, p3) {
                    return (p1.x - p3.x) * (p2.y - p3.y) - (p2.x - p3.x) * (p1.y - p3.y);
                }
                const b1 = sign(p, v[0], v[1]) < 0;
                const b2 = sign(p, v[1], v[2]) < 0;
                const b3 = sign(p, v[2], v[0]) < 0;
                return ((b1 == b2) && (b2 == b3));
            }

            function checkQuad(x, y) {
                // Quad corner coordinates
                const corners = [
                    {x, y}, {x: x + 1, y}, {x, y: y + 1}, {x: x + 1, y: y + 1}
                ];
                
                let coveredPoints = 0;
                corners.forEach(p => {
                    if (isInsideTriangle(p.x, p.y)) {
                        coveredPoints++;
                    }
                });

                // Simple coverage check: if at least one corner is inside
                return coveredPoints > 0;
            }

            function getGridCellIndex(x, y) {
                return y * GRID_SIZE + x;
            }

            function startDetailedScanConverterSimulation() {
                if (scSimulationInterval) return;

                // Reset state
                scOutputEl.innerHTML = '';
                currentQuadCount = 0;
                scQuadCountEl.textContent = 0;
                scanPosition = {x: 0, y: 0};
                scStatusEl.textContent = '开始扫描...';
                scSimulateBtn.disabled = true;
                
                scRasterGridEl.querySelector('.sc-triangle-poly').style.clipPath = `polygon(
                    ${TRIANGLE_VERTICES[0].x * 10}% ${TRIANGLE_VERTICES[0].y * 10}%,
                    ${TRIANGLE_VERTICES[1].x * 10}% ${TRIANGLE_VERTICES[1].y * 10}%,
                    ${TRIANGLE_VERTICES[2].x * 10}% ${TRIANGLE_VERTICES[2].y * 10}%
                )`;
                
                // Create a marker to show the scanning position
                const marker = document.createElement('div');
                marker.classList.add('sc-scanner-marker');
                scRasterGridEl.appendChild(marker);

                let x = 0;
                let y = 0;
                scSimulationInterval = setInterval(() => {
                    if (y >= GRID_SIZE - 1) {
                        clearInterval(scSimulationInterval);
                        scStatusEl.textContent = `光栅化完成！共生成 ${currentQuadCount} 个四边形。`;
                        scSimulateBtn.disabled = false;
                        marker.remove();
                        return;
                    }
                    
                    const markerX = (x * CELL_SIZE) + (CELL_SIZE);
                    const markerY = (y * CELL_SIZE) + (CELL_SIZE);
                    marker.style.transform = `translate(${markerX}px, ${markerY}px)`;
                    scStatusEl.textContent = `扫描位置: (${x}, ${y})`;
                    
                    const quadCovered = checkQuad(x, y);
                    if (quadCovered) {
                        // Highlight the 2x2 quad
                        const cells = [
                            scRasterGridEl.children[getGridCellIndex(x, y)],
                            scRasterGridEl.children[getGridCellIndex(x+1, y)],
                            scRasterGridEl.children[getGridCellIndex(x, y+1)],
                            scRasterGridEl.children[getGridCellIndex(x+1, y+1)]
                        ];
                        cells.forEach(cell => {
                            if(cell) {
                                cell.style.backgroundColor = '#facc15';
                            }
                        });

                        // Add quad to output
                        const quad = document.createElement('div');
                        quad.classList.add('pixel-quad');
                        scOutputEl.appendChild(quad);
                        currentQuadCount++;
                        scQuadCountEl.textContent = currentQuadCount;
                    }
                    
                    x += 2; // Move to the next quad column
                    if (x >= GRID_SIZE - 1) {
                        x = 0;
                        y += 2; // Move to the next quad row
                    }
                }, 100); // Animation speed
            }

            function resetScanConverterSimulation() {
                clearInterval(scSimulationInterval);
                scSimulationInterval = null;
                scRasterGridEl.querySelectorAll('.sc-scanner-marker').forEach(m => m.remove());
                scOutputEl.innerHTML = '';
                currentQuadCount = 0;
                scQuadCountEl.textContent = 0;
                scStatusEl.textContent = '点击“开始”按钮';
                scSimulateBtn.disabled = false;
                scRasterGridEl.querySelectorAll('.sc-grid-cell').forEach(cell => cell.style.backgroundColor = 'rgba(255, 255, 255, 0.1)');
            }
            
            createRasterGrid();
            scSimulateBtn.addEventListener('click', startDetailedScanConverterSimulation);
            scResetBtn.addEventListener('click', resetScanConverterSimulation);
            
            // --- End of New Detailed Scan Converter Simulation Logic ---

            // --- Async Compute Logic ---
            const toggleAsyncBtn = document.getElementById('toggle-async-btn');
            const asyncFill = document.getElementById('async-fill');
            let asyncEnabled = false;
            toggleAsyncBtn.addEventListener('click', () => {
                asyncEnabled = !asyncEnabled;
                if (asyncEnabled) {
                    asyncFill.style.width = '20%';
                    toggleAsyncBtn.textContent = '禁用异步计算';
                    toggleAsyncBtn.classList.remove('bg-slate-800', 'hover:bg-slate-700');
                    toggleAsyncBtn.classList.add('bg-amber-500', 'hover:bg-amber-600');
                } else {
                    asyncFill.style.width = '0%';
                    toggleAsyncBtn.textContent = '启用异步计算';
                    toggleAsyncBtn.classList.remove('bg-amber-500', 'hover:bg-amber-600');
                    toggleAsyncBtn.classList.add('bg-slate-800', 'hover:bg-slate-700');
                }
            });
            
            // --- Occupancy Chart Logic ---
            const vgprSlider = document.getElementById('vgpr-slider');
            const vgprValue = document.getElementById('vgpr-value');
            const ctx = document.getElementById('occupancyChart').getContext('2d');
            const maxVgprsPerCu = 65536;
            const maxWavefrontsPerCu = 40; 
            const occupancyChart = new Chart(ctx, {
                type: 'bar',
                data: {
                    labels: ['活动波前数量', 'CU 占用率'],
                    datasets: [{
                        label: '性能指标',
                        data: [0, 0],
                        backgroundColor: [
                            'rgba(59, 130, 246, 0.6)',
                            'rgba(251, 191, 36, 0.6)'
                        ],
                        borderColor: [
                            'rgba(59, 130, 246, 1)',
                            'rgba(251, 191, 36, 1)'
                        ],
                        borderWidth: 1
                    }]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                    indexAxis: 'y',
                    scales: {
                        x: {
                            beginAtZero: true,
                            max: 100,
                            ticks: {
                                callback: function(value) {
                                    return value + '% / 个'
                                }
                            }
                        }
                    },
                    plugins: {
                        legend: { display: false },
                        tooltip: {
                             callbacks: {
                                label: function(context) {
                                    let label = context.dataset.label || '';
                                    if (label) {
                                        label += ': ';
                                    }
                                    if (context.parsed.x !== null) {
                                        if (context.dataIndex === 0) {
                                            label += `${context.parsed.x.toFixed(0)} 个`;
                                        } else {
                                            label += `${context.parsed.x.toFixed(1)}%`;
                                        }
                                    }
                                    return label;
                                }
                            }
                        }
                    }
                }
            });

            function updateOccupancyChart(vgprs) {
                const vgprsPerWavefront = vgprs * 64;
                const maxWavefrontsByVgpr = Math.floor(maxVgprsPerCu / vgprsPerWavefront);
                const activeWavefronts = Math.min(maxWavefrontsPerCu, maxWavefrontsByVgpr);
                const occupancy = (activeWavefronts / maxWavefrontsPerCu) * 100;
                occupancyChart.data.datasets[0].data[0] = activeWavefronts;
                occupancyChart.data.datasets[0].data[1] = occupancy;
                occupancyChart.options.scales.x.max = Math.max(40, activeWavefronts) * 1.1;
                occupancyChart.update();
                vgprValue.textContent = vgprs;
            }

            vgprSlider.addEventListener('input', (e) => {
                updateOccupancyChart(parseInt(e.target.value));
            });
            updateOccupancyChart(parseInt(vgprSlider.value));


            // --- Dynamic CU Simulation Logic ---
            const NUM_CUS = 16;
            const TOTAL_VERTICES = 2560;
            const VERTEX_WAVEFRONT_SIZE = 64;
            const PIXEL_WAVEFRONT_SIZE = 64;
            const VERTEX_SHADER_DURATION = 15; // Time ticks
            const PIXEL_SHADER_DURATION = 10;  // Time ticks
            const PIXEL_WAVEFRONTS_PER_VERTEX_WAVEFRONT = 2; // Simple ratio for simulation

            const cuGrid = document.getElementById('cu-grid');
            const startBtn = document.getElementById('start-simulation-btn');
            const resetBtn = document.getElementById('reset-simulation-btn');
            const vertexTasksLeftEl = document.getElementById('vertex-tasks-left');
            const pixelTasksLeftEl = document.getElementById('pixel-tasks-left');
            const tasksCompletedEl = document.getElementById('tasks-completed');

            let cuStates = [];
            let vertexWavefronts = 0;
            let pixelWavefronts = 0;
            let processedVertexWavefronts = 0;
            let processedPixelWavefronts = 0;
            let totalTasks = 0;
            let simulationInterval_cu = null;

            function createCuElements() {
                cuGrid.innerHTML = '';
                for (let i = 0; i < NUM_CUS; i++) {
                    const cuDiv = document.createElement('div');
                    cuDiv.classList.add('cu-box');
                    cuDiv.setAttribute('id', `cu-${i}`);
                    cuDiv.innerHTML = `<div class="content">CU ${i}: 空闲</div>`;
                    cuGrid.appendChild(cuDiv);
                }
            }

            function initializeSimulation_cu() {
                cuStates = Array(NUM_CUS).fill(null).map((_, i) => ({
                    id: i,
                    type: 'idle',
                    remainingTime: 0
                }));
                vertexWavefronts = TOTAL_VERTICES / VERTEX_WAVEFRONT_SIZE; // 40
                pixelWavefronts = 0;
                processedVertexWavefronts = 0;
                processedPixelWavefronts = 0;
                totalTasks = vertexWavefronts + (vertexWavefronts * PIXEL_WAVEFRONTS_PER_VERTEX_WAVEFRONT);
                
                startBtn.disabled = false;
                startBtn.textContent = '开始模拟';
                updateUI_cu();
            }

            function updateUI_cu() {
                cuStates.forEach(cu => {
                    const el = document.getElementById(`cu-${cu.id}`);
                    el.className = 'cu-box';
                    let content = `<div class="content">CU ${cu.id}: 空闲</div>`;
                    if (cu.type === 'vertex') {
                        el.classList.add('vertex');
                        content = `<div class="content">CU ${cu.id}: 顶点着色<div class="progress-bar"><div class="progress" style="width: ${((VERTEX_SHADER_DURATION - cu.remainingTime) / VERTEX_SHADER_DURATION) * 100}%"></div></div></div>`;
                    } else if (cu.type === 'pixel') {
                        el.classList.add('pixel');
                        content = `<div class="content">CU ${cu.id}: 像素着色<div class="progress-bar"><div class="progress" style="width: ${((PIXEL_SHADER_DURATION - cu.remainingTime) / PIXEL_SHADER_DURATION) * 100}%"></div></div></div>`;
                    }
                    el.innerHTML = content;
                });
                vertexTasksLeftEl.textContent = vertexWavefronts;
                pixelTasksLeftEl.textContent = pixelWavefronts;
                tasksCompletedEl.textContent = processedVertexWavefronts + processedPixelWavefronts;
            }

            function simulationTick_cu() {
                // Step 1: Process running tasks
                cuStates.forEach(cu => {
                    if (cu.type !== 'idle') {
                        cu.remainingTime--;
                        if (cu.remainingTime <= 0) {
                            if (cu.type === 'vertex') {
                                // A vertex wavefront is finished, generate pixel wavefronts
                                pixelWavefronts += PIXEL_WAVEFRONTS_PER_VERTEX_WAVEFRONT;
                                processedVertexWavefronts++;
                            } else if (cu.type === 'pixel') {
                                processedPixelWavefronts++;
                            }
                            // CU becomes idle again
                            cu.type = 'idle';
                        }
                    }
                });

                // Step 2: Dispatch new tasks to idle CUs
                cuStates.forEach(cu => {
                    if (cu.type === 'idle') {
                        // Prioritize vertex shaders if available
                        if (vertexWavefronts > 0) {
                            cu.type = 'vertex';
                            cu.remainingTime = VERTEX_SHADER_DURATION;
                            vertexWavefronts--;
                        } else if (pixelWavefronts > 0) {
                            // If no more vertex shaders, schedule pixel shaders
                            cu.type = 'pixel';
                            cu.remainingTime = PIXEL_SHADER_DURATION;
                            pixelWavefronts--;
                        }
                    }
                });

                updateUI_cu();
                
                // Step 3: Check for completion
                if (processedVertexWavefronts + processedPixelWavefronts >= totalTasks) {
                    clearInterval(simulationInterval_cu);
                    simulationInterval_cu = null;
                    startBtn.textContent = '模拟完成！';
                    startBtn.disabled = true;
                }
            }

            startBtn.addEventListener('click', () => {
                if (simulationInterval_cu) {
                    clearInterval(simulationInterval_cu);
                    simulationInterval_cu = null;
                    startBtn.textContent = '继续模拟';
                } else {
                    simulationInterval_cu = setInterval(simulationTick_cu, 100);
                    startBtn.textContent = '暂停模拟';
                }
            });

            resetBtn.addEventListener('click', () => {
                clearInterval(simulationInterval_cu);
                simulationInterval_cu = null;
                initializeSimulation_cu();
            });

            createCuElements();
            initializeSimulation_cu();
            
            // --- Click-to-show CU component details logic ---
            const cuComponents = document.querySelectorAll('.cu-diagram .component');
            const cuDetailsEl = document.getElementById('cu-details');

            cuComponents.forEach(component => {
                component.addEventListener('click', () => {
                    const info = component.getAttribute('data-info');
                    if (info) {
                        cuDetailsEl.textContent = info;
                        cuDetailsEl.classList.remove('hidden');
                    } else {
                        cuDetailsEl.textContent = '点击上方的组件以查看详细信息。';
                        cuDetailsEl.classList.remove('hidden');
                    }
                });
            });

        });
    </script>
</body>
</html>
</div>