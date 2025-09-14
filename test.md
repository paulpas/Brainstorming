```mermaid
<svg width="1400" height="1200" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#333" />
    </marker>
    
    <!-- Gradients for different layers -->
    <linearGradient id="userGrad" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#e1f5fe;stop-opacity:1" />
      <stop offset="100%" style="stop-color:#b3e5fc;stop-opacity:1" />
    </linearGradient>
    
    <linearGradient id="coreGrad" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#f3e5f5;stop-opacity:1" />
      <stop offset="100%" style="stop-color:#e1bee7;stop-opacity:1" />
    </linearGradient>
    
    <linearGradient id="aiGrad" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#e8f5e8;stop-opacity:1" />
      <stop offset="100%" style="stop-color:#c8e6c9;stop-opacity:1" />
    </linearGradient>
    
    <linearGradient id="autoGrad" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#fff3e0;stop-opacity:1" />
      <stop offset="100%" style="stop-color:#ffcc02;stop-opacity:1" />
    </linearGradient>
    
    <linearGradient id="dataGrad" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#fce4ec;stop-opacity:1" />
      <stop offset="100%" style="stop-color:#f8bbd9;stop-opacity:1" />
    </linearGradient>
  </defs>
  
  <!-- Background -->
  <rect width="1400" height="1200" fill="#fafafa"/>
  
  <!-- Title -->
  <text x="700" y="30" text-anchor="middle" font-family="Arial, sans-serif" font-size="24" font-weight="bold" fill="#333">
    WorkflowAI - Cross-App Automation Architecture
  </text>
  
  <!-- User Layer -->
  <g id="userLayer">
    <rect x="50" y="60" width="120" height="80" rx="10" fill="url(#userGrad)" stroke="#01579b" stroke-width="2"/>
    <text x="110" y="85" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" fill="#01579b">👤 User</text>
    <text x="110" y="120" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" fill="#01579b">Interaction</text>
    
    <rect x="200" y="60" width="120" height="80" rx="10" fill="url(#userGrad)" stroke="#01579b" stroke-width="2"/>
    <text x="260" y="85" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" fill="#01579b">📱 Android</text>
    <text x="260" y="105" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" fill="#01579b">Phone</text>
  </g>
  
  <!-- Input Capture Layer -->
  <g id="inputLayer">
    <rect x="50" y="180" width="120" height="60" rx="8" fill="url(#coreGrad)" stroke="#4a148c" stroke-width="2"/>
    <text x="110" y="200" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#4a148c">👆 Touch</text>
    <text x="110" y="220" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#4a148c">Events</text>
    
    <rect x="200" y="180" width="120" height="60" rx="8" fill="url(#coreGrad)" stroke="#4a148c" stroke-width="2"/>
    <text x="260" y="200" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#4a148c">🖥️ Screen</text>
    <text x="260" y="220" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#4a148c">State</text>
    
    <rect x="350" y="180" width="120" height="60" rx="8" fill="url(#coreGrad)" stroke="#4a148c" stroke-width="2"/>
    <text x="410" y="200" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#4a148c">📋 Clipboard</text>
    <text x="410" y="220" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#4a148c">Data</text>
  </g>
  
  <!-- Core Processing Engine -->
  <g id="coreEngine">
    <rect x="550" y="160" width="300" height="100" rx="12" fill="url(#coreGrad)" stroke="#4a148c" stroke-width="3"/>
    <text x="700" y="185" text-anchor="middle" font-family="Arial, sans-serif" font-size="14" font-weight="bold" fill="#4a148c">🧠 Behavior Analysis Engine</text>
    
    <rect x="570" y="200" width="80" height="25" rx="5" fill="#e1bee7" stroke="#4a148c"/>
    <text x="610" y="215" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#4a148c">Pattern Detection</text>
    
    <rect x="660" y="200" width="80" height="25" rx="5" fill="#e1bee7" stroke="#4a148c"/>
    <text x="700" y="215" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#4a148c">Sequence Mining</text>
    
    <rect x="750" y="200" width="80" height="25" rx="5" fill="#e1bee7" stroke="#4a148c"/>
    <text x="790" y="215" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#4a148c">App Transitions</text>
    
    <rect x="615" y="230" width="80" height="25" rx="5" fill="#e1bee7" stroke="#4a148c"/>
    <text x="655" y="245" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#4a148c">Watch Mode</text>
    
    <rect x="705" y="230" width="80" height="25" rx="5" fill="#e1bee7" stroke="#4a148c"/>
    <text x="745" y="245" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#4a148c">Learning</text>
  </g>
  
  <!-- AI Processing Layer -->
  <g id="aiLayer">
    <rect x="950" y="160" width="300" height="100" rx="12" fill="url(#aiGrad)" stroke="#2e7d32" stroke-width="3"/>
    <text x="1100" y="185" text-anchor="middle" font-family="Arial, sans-serif" font-size="14" font-weight="bold" fill="#2e7d32">🤖 Local LLM Processor</text>
    
    <rect x="970" y="200" width="85" height="25" rx="5" fill="#c8e6c9" stroke="#2e7d32"/>
    <text x="1012" y="215" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#2e7d32">Data Extraction</text>
    
    <rect x="1065" y="200" width="85" height="25" rx="5" fill="#c8e6c9" stroke="#2e7d32"/>
    <text x="1107" y="215" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#2e7d32">Semantic Analysis</text>
    
    <rect x="1160" y="200" width="85" height="25" rx="5" fill="#c8e6c9" stroke="#2e7d32"/>
    <text x="1202" y="215" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#2e7d32">Data Transform</text>
    
    <rect x="1017" y="230" width="85" height="25" rx="5" fill="#c8e6c9" stroke="#2e7d32"/>
    <text x="1059" y="245" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#2e7d32">Context Understanding</text>
    
    <rect x="1112" y="230" width="85" height="25" rx="5" fill="#c8e6c9" stroke="#2e7d32"/>
    <text x="1154" y="245" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#2e7d32">Workflow Discovery</text>
  </g>
  
  <!-- Workflow Management -->
  <g id="workflowMgmt">
    <rect x="200" y="320" width="150" height="80" rx="10" fill="url(#autoGrad)" stroke="#e65100" stroke-width="2"/>
    <text x="275" y="345" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" fill="#e65100">💡 Macro</text>
    <text x="275" y="365" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" fill="#e65100">Suggestions</text>
    <text x="275" y="385" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#e65100">&amp; Builder</text>
    
    <rect x="400" y="320" width="150" height="80" rx="10" fill="url(#autoGrad)" stroke="#e65100" stroke-width="2"/>
    <text x="475" y="345" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" fill="#e65100">🗺️ Cross-App</text>
    <text x="475" y="365" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" fill="#e65100">Workflow</text>
    <text x="475" y="385" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#e65100">Mapper</text>
  </g>
  
  <!-- Automation Engine -->
  <g id="automationEngine">
    <rect x="600" y="320" width="200" height="80" rx="12" fill="url(#autoGrad)" stroke="#e65100" stroke-width="3"/>
    <text x="700" y="345" text-anchor="middle" font-family="Arial, sans-serif" font-size="14" font-weight="bold" fill="#e65100">⚙️ Automation Engine</text>
    
    <rect x="620" y="365" width="60" height="20" rx="3" fill="#ffcc02" stroke="#e65100"/>
    <text x="650" y="377" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="#e65100">Gesture Replay</text>
    
    <rect x="690" y="365" width="60" height="20" rx="3" fill="#ffcc02" stroke="#e65100"/>
    <text x="720" y="377" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="#e65100">Data Pipeline</text>
    
    <rect x="760" y="365" width="60" height="20" rx="3" fill="#ffcc02" stroke="#e65100"/>
    <text x="790" y="377" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="#e65100">Orchestrator</text>
  </g>
  
  <!-- Data Storage -->
  <g id="dataStorage">
    <rect x="850" y="320" width="120" height="60" rx="8" fill="url(#dataGrad)" stroke="#880e4f" stroke-width="2"/>
    <text x="910" y="340" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#880e4f">🗄️ Behavior</text>
    <text x="910" y="360" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#880e4f">Database</text>
    
    <rect x="1000" y="320" width="120" height="60" rx="8" fill="url(#dataGrad)" stroke="#880e4f" stroke-width="2"/>
    <text x="1060" y="340" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#880e4f">📚 Workflow</text>
    <text x="1060" y="360" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#880e4f">Database</text>
    
    <rect x="1150" y="320" width="120" height="60" rx="8" fill="url(#dataGrad)" stroke="#880e4f" stroke-width="2"/>
    <text x="1210" y="340" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#880e4f">📝 Execution</text>
    <text x="1210" y="360" text-anchor="middle" font-family="Arial, sans-serif" font-size="11" fill="#880e4f">Log</text>
  </g>
  
  <!-- User Interface -->
  <g id="userInterface">
    <rect x="50" y="450" width="200" height="100" rx="12" fill="url(#userGrad)" stroke="#01579b" stroke-width="3"/>
    <text x="150" y="475" text-anchor="middle" font-family="Arial, sans-serif" font-size="14" font-weight="bold" fill="#01579b">📊 User Dashboard</text>
    
    <rect x="70" y="490" width="50" height="25" rx="5" fill="#b3e5fc" stroke="#01579b"/>
    <text x="95" y="505" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#01579b">Workflows</text>
    
    <rect x="130" y="490" width="50" height="25" rx="5" fill="#b3e5fc" stroke="#01579b"/>
    <text x="155" y="505" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#01579b">Schedule</text>
    
    <rect x="190" y="490" width="50" height="25" rx="5" fill="#b3e5fc" stroke="#01579b"/>
    <text x="215" y="505" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#01579b">Insights</text>
    
    <rect x="100" y="520" width="100" height="25" rx="5" fill="#b3e5fc" stroke="#01579b"/>
    <text x="150" y="535" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#01579b">Performance Analytics</text>
  </g>
  
  <!-- Execution Modes -->
  <g id="executionModes">
    <rect x="300" y="450" width="100" height="40" rx="8" fill="#e8f5e8" stroke="#2e7d32" stroke-width="2"/>
    <text x="350" y="465" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="#2e7d32">🌙 Night Mode</text>
    <text x="350" y="480" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="#2e7d32">Execution</text>
    
    <rect x="300" y="500" width="100" height="40" rx="8" fill="#e8f5e8" stroke="#2e7d32" stroke-width="2"/>
    <text x="350" y="515" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="#2e7d32">😴 Idle Mode</text>
    <text x="350" y="530" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="#2e7d32">Execution</text>
  </g>
  
  <!-- Cross-App Example -->
  <g id="crossAppExample">
    <rect x="450" y="450" width="400" height="150" rx="12" fill="#f5f5f5" stroke="#666" stroke-width="2" stroke-dasharray="5,5"/>
    <text x="650" y="470" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" font-weight="bold" fill="#666">Cross-App Workflow Example</text>
    
    <!-- Banking App -->
    <rect x="470" y="480" width="70" height="35" rx="5" fill="#4caf50" stroke="#2e7d32"/>
    <text x="505" y="492" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">🏦 Bank</text>
    <text x="505" y="505" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">$15,234</text>
    
    <!-- Arrow 1 -->
    <path d="M540 497 L560 497" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
    
    <!-- Transform -->
    <rect x="565" y="485" width="50" height="25" rx="5" fill="#ff9800" stroke="#e65100"/>
    <text x="590" y="500" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">Format</text>
    
    <!-- Arrow 2 -->
    <path d="M615 497 L635 497" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
    
    <!-- Stock App -->
    <rect x="640" y="480" width="70" height="35" rx="5" fill="#2196f3" stroke="#1976d2"/>
    <text x="675" y="492" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">📈 Stocks</text>
    <text x="675" y="505" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">$43,567</text>
    
    <!-- Arrow 3 -->
    <path d="M710 497 L730 497" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
    
    <!-- Calculate -->
    <rect x="735" y="485" width="50" height="25" rx="5" fill="#9c27b0" stroke="#7b1fa2"/>
    <text x="760" y="500" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">Calculate</text>
    
    <!-- Arrow 4 -->
    <path d="M760 510 L760 530" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
    
    <!-- Final Output -->
    <rect x="720" y="535" width="80" height="35" rx="5" fill="#795548" stroke="#5d4037"/>
    <text x="760" y="547" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">💬 Summary</text>
    <text x="760" y="560" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">$58.8K Total</text>
    
    <!-- Crypto App -->
    <rect x="470" y="525" width="70" height="35" rx="5" fill="#ff5722" stroke="#d84315"/>
    <text x="505" y="537" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">₿ Crypto</text>
    <text x="505" y="550" text-anchor="middle" font-family="Arial, sans-serif" font-size="8" fill="white">$8,921</text>
    
    <!-- Crypto to Calculate arrow -->
    <path d="M540 542 Q650 560 735 520" stroke="#333" stroke-width="2" marker-end="url(#arrowhead)" fill="none"/>
  </g>
  
  <!-- Security & Privacy -->
  <g id="security">
    <rect x="900" y="450" width="150" height="100" rx="10" fill="#fff8e1" stroke="#f57c00" stroke-width="2"/>
    <text x="975" y="470" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" font-weight="bold" fill="#f57c00">🔐 Security Layer</text>
    
    <text x="975" y="490" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="#f57c00">🔒 Local Processing</text>
    <text x="975" y="505" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="#f57c00">🔐 Encrypted Storage</text>
    <text x="975" y="520" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="#f57c00">🚫 No Cloud Sync</text>
    <text x="975" y="535" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="#f57c00">👤 User Control</text>
  </g>
  
  <!-- Watch Mode Detail -->
  <g id="watchModeDetail">
    <rect x="1100" y="450" width="250" height="120" rx="12" fill="#e3f2fd" stroke="#1976d2" stroke-width="2" stroke-dasharray="3,3"/>
    <text x="1225" y="470" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" font-weight="bold" fill="#1976d2">👁️ Watch Mode Learning</text>
    
    <rect x="1120" y="480" width="80" height="25" rx="5" fill="#bbdefb" stroke="#1976d2"/>
    <text x="1160" y="495" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#1976d2">Observe Actions</text>
    
    <path d="M1200 492 L1220 492" stroke="#1976d2" stroke-width="2" marker-end="url(#arrowhead)"/>
    
    <rect x="1225" y="480" width="80" height="25" rx="5" fill="#bbdefb" stroke="#1976d2"/>
    <text x="1265" y="495" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#1976d2">Detect Patterns</text>
    
    <path d="M1265 505 L1265 525" stroke="#1976d2" stroke-width="2" marker-end="url(#arrowhead)"/>
    
    <rect x="1225" y="530" width="80" height="25" rx="5" fill="#bbdefb" stroke="#1976d2"/>
    <text x="1265" y="545" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#1976d2">Suggest Macros</text>
    
    <path d="M1225 542 L1205 542" stroke="#1976d2" stroke-width="2" marker-end="url(#arrowhead)"/>
    
    <rect x="1120" y="530" width="80" height="25" rx="5" fill="#bbdefb" stroke="#1976d2"/>
    <text x="1160" y="545" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#1976d2">User Validates</text>
  </g>
  
  <!-- Key Data Flow Arrows -->
  <!-- User to Phone -->
  <path d="M170 100 L200 100" stroke="#01579b" stroke-width="2" marker-end="url(#arrowhead)"/>
  
  <!-- Phone to Inputs -->
  <path d="M260 140 L260 160 Q260 170 250 170 L180 170 Q170 170 170 180 L170 190" stroke="#4a148c" stroke-width="2" marker-end="url(#arrowhead)" fill="none"/>
  <path d="M260 140 L260 180" stroke="#4a148c" stroke-width="2" marker-end="url(#arrowhead)"/>
  <path d="M260 140 L260 160 Q260 170 270 170 L380 170 Q390 170 390 180 L390 190" stroke="#4a148c" stroke-width="2" marker-end="url(#arrowhead)" fill="none"/>
  
  <!-- Inputs to Core Engine -->
  <path d="M170 210 L550 210" stroke="#4a148c" stroke-width="3" marker-end="url(#arrowhead)"/>
  <path d="M320 210 L550 210" stroke="#4a148c" stroke-width="3"/>
  <path d="M470 210 L550 210" stroke="#4a148c" stroke-width="3"/>
  
  <!-- Core to AI -->
  <path d="M850 210 L950 210" stroke="#2e7d32" stroke-width="3" marker-end="url(#arrowhead)"/>
  
  <!-- AI to Workflow -->
  <path d="M1100 260 Q1100 290 700 290 Q400 290 400 320" stroke="#e65100" stroke-width="2" marker-end="url(#arrowhead)" fill="none"/>
  
  <!-- Workflow to Automation -->
  <path d="M550 360 L600 360" stroke="#e65100" stroke-width="3" marker-end="url(#arrowhead)"/>
  
  <!-- Automation to Storage -->
  <path d="M800 350 L850 350" stroke="#880e4f" stroke-width="2" marker-end="url(#arrowhead)"/>
  <path d="M800 350 L1000 350" stroke="#880e4f" stroke-width="2"/>
  <path d="M800 350 L1150 350" stroke="#880e4f" stroke-width="2"/>
  
  <!-- Dashboard connections -->
  <path d="M250 500 L300 500" stroke="#01579b" stroke-width="2" marker-end="url(#arrowhead)"/>
  <path d="M150 450 Q150 420 275 420 Q400 420 400 400" stroke="#01579b" stroke-width="2" marker-end="url(#arrowhead)" fill="none"/>
  
  <!-- Performance feedback loop -->
  <path d="M1210 380 Q1210 420 1100 420 Q700 420 700 400" stroke="#2e7d32" stroke-width="2" marker-end="url(#arrowhead)" fill="none" stroke-dasharray="3,3"/>
  
  <!-- Legend -->
  <g id="legend" transform="translate(50, 650)">
    <rect x="0" y="0" width="300" height="180" rx="8" fill="#f9f9f9" stroke="#666" stroke-width="1"/>
    <text x="150" y="20" text-anchor="middle" font-family="Arial, sans-serif" font-size="14" font-weight="bold" fill="#333">Component Legend</text>
    
    <rect x="10" y="30" width="20" height="15" fill="url(#userGrad)" stroke="#01579b"/>
    <text x="35" y="42" font-family="Arial, sans-serif" font-size="11" fill="#333">User Interface Layer</text>
    
    <rect x="10" y="50" width="20" height="15" fill="url(#coreGrad)" stroke="#4a148c"/>
    <text x="35" y="62" font-family="Arial, sans-serif" font-size="11" fill="
```
