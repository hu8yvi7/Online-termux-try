#!/usr/bin/env python3
"""
Termux Web Terminal Pro - Complete Web-based Terminal
- All Termux commands support
- Real-time output streaming
- File upload/download
- Process management with /stop
- Live logs display
- File manager
"""
import os
import sys
import subprocess
import threading
import time
import webbrowser
import json
import signal
import psutil
import shutil
from datetime import datetime
from pathlib import Path
from werkzeug.utils import secure_filename
from flask import Flask, request, jsonify, render_template_string, Response, send_file

app = Flask(__name__)
app.config['MAX_CONTENT_LENGTH'] = 100 * 1024 * 1024  # 100MB max file size

# Global state
class TerminalState:
    def __init__(self):
        self.current_dir = os.getcwd()
        self.processes = {}  # pid -> {'process': obj, 'cmd': str, 'type': 'python/bash'}
        self.command_history = []
        self.logs = []
        self.upload_dir = os.path.join(os.getcwd(), 'uploads')
        os.makedirs(self.upload_dir, exist_ok=True)

state = TerminalState()

# ==================== UTILITY FUNCTIONS ====================
def log_message(msg, level="INFO"):
    timestamp = datetime.now().strftime("%H:%M:%S")
    log_entry = f"[{timestamp}] [{level}] {msg}"
    state.logs.append(log_entry)
    if len(state.logs) > 1000:  # Keep last 1000 logs
        state.logs = state.logs[-1000:]
    print(log_entry)
    return log_entry

def kill_process_tree(pid):
    """Kill process and all children"""
    try:
        parent = psutil.Process(pid)
        children = parent.children(recursive=True)
        
        for child in children:
            try:
                child.kill()
            except:
                pass
        
        try:
            parent.kill()
        except:
            pass
        
        return True
    except psutil.NoSuchProcess:
        return False
    except Exception as e:
        log_message(f"Error killing process {pid}: {e}", "ERROR")
        return False

def get_all_processes():
    """Get all processes started from this terminal"""
    procs = []
    for pid, info in state.processes.items():
        try:
            proc = psutil.Process(pid)
            procs.append({
                'pid': pid,
                'cmd': info['cmd'],
                'status': 'running' if proc.is_running() else 'dead',
                'start_time': datetime.fromtimestamp(proc.create_time()).strftime("%H:%M:%S"),
                'type': info.get('type', 'unknown')
            })
        except:
            procs.append({
                'pid': pid,
                'cmd': info['cmd'],
                'status': 'dead',
                'start_time': 'unknown',
                'type': info.get('type', 'unknown')
            })
    return procs

def execute_command_stream(cmd, cwd):
    """Execute command and stream output"""
    log_message(f"Executing: {cmd}", "COMMAND")
    
    try:
        process = subprocess.Popen(
            cmd,
            shell=True,
            cwd=cwd,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True,
            bufsize=1,
            universal_newlines=True
        )
        
        pid = process.pid
        state.processes[pid] = {
            'process': process,
            'cmd': cmd,
            'type': 'python' if 'python' in cmd.lower() else 'bash'
        }
        
        # Read output line by line
        for line in iter(process.stdout.readline, ''):
            if line:
                yield line.rstrip() + "\n"
        
        process.wait()
        
        # Remove from active processes
        if pid in state.processes:
            del state.processes[pid]
            
        yield f"\n[Process completed with exit code: {process.returncode}]\n"
        
    except Exception as e:
        yield f"Error executing command: {str(e)}\n"

# ==================== HTML TEMPLATES ====================
INDEX_HTML = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Termux Web Terminal Pro</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            background: #0a0a0a;
            color: #00ff00;
            font-family: 'Courier New', monospace;
            height: 100vh;
            overflow: hidden;
        }
        .header {
            background: #111;
            padding: 10px 15px;
            border-bottom: 2px solid #00ffff;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .header h1 { color: #00ffff; font-size: 22px; }
        .status-bar {
            display: flex;
            gap: 15px;
            font-size: 14px;
        }
        .status-item { padding: 3px 8px; background: #222; border-radius: 3px; }
        .container {
            display: flex;
            height: calc(100vh - 120px);
        }
        .sidebar {
            width: 300px;
            background: #111;
            border-right: 2px solid #333;
            padding: 10px;
            overflow-y: auto;
        }
        .main-content {
            flex: 1;
            display: flex;
            flex-direction: column;
        }
        .terminal-container {
            flex: 1;
            padding: 10px;
            overflow-y: auto;
            background: #000;
        }
        .terminal-line {
            white-space: pre-wrap;
            word-break: break-all;
            line-height: 1.4;
            margin: 2px 0;
            padding: 2px 5px;
        }
        .prompt { color: #00ffff; font-weight: bold; }
        .command { color: #ffaa00; }
        .output { color: #cccccc; }
        .error { color: #ff5555; }
        .success { color: #55ff55; }
        .info { color: #5555ff; }
        .input-area {
            background: #111;
            padding: 10px;
            border-top: 2px solid #333;
            display: flex;
            align-items: center;
            gap: 10px;
        }
        #command-input {
            flex: 1;
            background: #000;
            color: #00ff00;
            border: 1px solid #00ff00;
            padding: 8px;
            font-size: 14px;
            font-family: monospace;
        }
        #command-input:focus {
            outline: none;
            border-color: #00ffff;
            box-shadow: 0 0 5px #00ffff;
        }
        .btn {
            background: #0066cc;
            color: white;
            border: none;
            padding: 8px 15px;
            border-radius: 4px;
            cursor: pointer;
            font-weight: bold;
        }
        .btn:hover { background: #0088ff; }
        .btn-danger { background: #cc3300; }
        .btn-danger:hover { background: #ff5500; }
        .btn-success { background: #00aa00; }
        .btn-success:hover { background: #00cc00; }
        .controls {
            display: flex;
            gap: 8px;
            padding: 10px;
            background: #151515;
            flex-wrap: wrap;
        }
        .section {
            background: #151515;
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 5px;
        }
        .section h3 {
            color: #00ffff;
            margin-bottom: 8px;
            border-bottom: 1px solid #333;
            padding-bottom: 5px;
        }
        .process-item {
            background: #222;
            padding: 8px;
            margin: 5px 0;
            border-radius: 4px;
            border-left: 4px solid #00ff00;
        }
        .process-item.stopped { border-left-color: #ff5555; }
        .file-item {
            padding: 5px;
            cursor: pointer;
            border-radius: 3px;
        }
        .file-item:hover { background: #222; }
        .dir { color: #55aaff; }
        .file { color: #cccccc; }
        .log-entry {
            font-size: 12px;
            padding: 3px;
            border-bottom: 1px solid #333;
        }
        .tab-container {
            display: flex;
            background: #111;
            padding: 5px;
        }
        .tab {
            padding: 8px 15px;
            background: #222;
            margin-right: 2px;
            cursor: pointer;
            border-radius: 5px 5px 0 0;
        }
        .tab.active {
            background: #0066cc;
            border-bottom: 2px solid #00ffff;
        }
        .upload-area {
            border: 2px dashed #00ff00;
            padding: 20px;
            text-align: center;
            margin: 10px 0;
            cursor: pointer;
        }
        .upload-area:hover { background: #222; }
    </style>
</head>
<body>
    <div class="header">
        <h1>🐧 Termux Web Terminal Pro</h1>
        <div class="status-bar">
            <span class="status-item" id="current-dir">📁 ~</span>
            <span class="status-item" id="process-count">🔄 0 processes</span>
            <span class="status-item" id="user-info">👤 user@localhost</span>
        </div>
    </div>

    <div class="tab-container">
        <div class="tab active" onclick="switchTab('terminal')">💻 Terminal</div>
        <div class="tab" onclick="switchTab('files')">📁 File Manager</div>
        <div class="tab" onclick="switchTab('processes')">📊 Processes</div>
        <div class="tab" onclick="switchTab('logs')">📋 Logs</div>
        <div class="tab" onclick="switchTab('upload')">⬆️ Upload</div>
    </div>

    <div class="container">
        <!-- Sidebar with commands help -->
        <div class="sidebar" id="sidebar">
            <div class="section">
                <h3>📚 Quick Commands</h3>
                <div style="display: flex; flex-direction: column; gap: 5px;">
                    <button class="btn" onclick="runCmd('pwd')">pwd</button>
                    <button class="btn" onclick="runCmd('ls -la')">ls -la</button>
                    <button class="btn" onclick="runCmd('python3 --version')">Python Version</button>
                    <button class="btn" onclick="runCmd('pip list')">Installed Packages</button>
                    <button class="btn" onclick="runCmd('neofetch')">System Info</button>
                    <button class="btn" onclick="runCmd('top -n 1')">Process List</button>
                </div>
            </div>
            
            <div class="section">
                <h3>⚡ Python Scripts</h3>
                <div id="python-files"></div>
            </div>
            
            <div class="section">
                <h3>🔧 Controls</h3>
                <div style="display: flex; flex-wrap: wrap; gap: 5px;">
                    <button class="btn-danger" onclick="sendCtrl('C')">Ctrl+C</button>
                    <button class="btn-danger" onclick="sendCtrl('Z')">Ctrl+Z</button>
                    <button class="btn-danger" onclick="killAllProcesses()">Kill All</button>
                    <button class="btn-success" onclick="clearTerminal()">Clear</button>
                    <button class="btn" onclick="refreshFiles()">Refresh</button>
                </div>
            </div>
            
            <div class="section">
                <h3>💡 Tips</h3>
                <div style="font-size: 12px; color: #888;">
                    • Type /stop filename.py to stop script<br>
                    • Click files to edit/run<br>
                    • Use arrow keys for history<br>
                    • Drag & drop files to upload<br>
                    • Ctrl+L to clear terminal
                </div>
            </div>
        </div>

        <!-- Main content area -->
        <div class="main-content">
            <!-- Terminal Tab -->
            <div id="terminal-tab" class="tab-content">
                <div class="controls">
                    <button class="btn" onclick="runCmd('cd /')">Root Dir</button>
                    <button class="btn" onclick="runCmd('cd ~')">Home Dir</button>
                    <button class="btn" onclick="runCmd('cd ..')">Up Directory</button>
                    <button class="btn" onclick="runCmd('clear')">Clear Terminal</button>
                    <button class="btn" onclick="runCmd('history')">Command History</button>
                    <button class="btn" onclick="showHelp()">Show Help</button>
                </div>
                
                <div class="terminal-container" id="terminal-output">
                    <div class="terminal-line success">╔════════════════════════════════════════╗</div>
                    <div class="terminal-line success">║   🐧 Termux Web Terminal Pro v1.0      ║</div>
                    <div class="terminal-line success">║   Complete Web-based Terminal         ║</div>
                    <div class="terminal-line success">╚════════════════════════════════════════╝</div>
                    <div class="terminal-line info">Type any Termux command below</div>
                    <div class="terminal-line info">Use /stop script.py to stop running scripts</div>
                    <div class="terminal-line info">Type 'help' for available commands</div>
                </div>
                
                <div class="input-area">
                    <span class="prompt">$</span>
                    <input type="text" id="command-input" placeholder="Type command and press Enter..." autofocus>
                    <button class="btn" onclick="executeCommand()">Execute</button>
                    <button class="btn-success" onclick="uploadFile()">📁 Upload</button>
                </div>
            </div>

            <!-- File Manager Tab -->
            <div id="files-tab" class="tab-content" style="display: none; padding: 20px; overflow-y: auto;">
                <div style="display: flex; justify-content: space-between; margin-bottom: 15px;">
                    <h2>📁 File Manager</h2>
                    <div>
                        <button class="btn" onclick="createNewFile()">📄 New File</button>
                        <button class="btn" onclick="createNewFolder()">📁 New Folder</button>
                        <button class="btn" onclick="refreshFileList()">🔄 Refresh</button>
                    </div>
                </div>
                <div style="margin-bottom: 15px;">
                    <input type="text" id="file-path" style="width: 80%; padding: 8px; background: #000; color: #fff; border: 1px solid #444;" value="/">
                    <button class="btn" onclick="changeFileDir()">Go</button>
                </div>
                <div id="file-list" style="background: #111; padding: 10px; border-radius: 5px; min-height: 300px;">
                    Loading files...
                </div>
            </div>

            <!-- Processes Tab -->
            <div id="processes-tab" class="tab-content" style="display: none; padding: 20px;">
                <h2>📊 Running Processes</h2>
                <div style="margin: 15px 0;">
                    <button class="btn-danger" onclick="killAllProcesses()">Kill All Processes</button>
                    <button class="btn" onclick="refreshProcesses()">Refresh List</button>
                </div>
                <div id="process-list" class="section">
                    Loading processes...
                </div>
            </div>

            <!-- Logs Tab -->
            <div id="logs-tab" class="tab-content" style="display: none; padding: 20px; overflow-y: auto;">
                <h2>📋 System Logs</h2>
                <div style="margin: 15px 0;">
                    <button class="btn" onclick="refreshLogs()">Refresh Logs</button>
                    <button class="btn" onclick="clearLogs()">Clear Logs</button>
                    <button class="btn" onclick="downloadLogs()">Download Logs</button>
                </div>
                <div id="logs-container" style="background: #000; padding: 10px; border-radius: 5px; font-family: monospace; font-size: 12px; max-height: 400px; overflow-y: auto;">
                    Loading logs...
                </div>
            </div>

            <!-- Upload Tab -->
            <div id="upload-tab" class="tab-content" style="display: none; padding: 20px;">
                <h2>⬆️ File Upload</h2>
                <div class="upload-area" onclick="document.getElementById('file-upload').click()" id="drop-area">
                    <div style="font-size: 24px;">📁</div>
                    <div>Click or drag & drop files here</div>
                    <div style="font-size: 12px; color: #888; margin-top: 10px;">Max file size: 100MB</div>
                </div>
                <input type="file" id="file-upload" multiple style="display: none;" onchange="handleFileSelect(event)">
                
                <div style="margin-top: 20px;">
                    <h3>Upload Queue</h3>
                    <div id="upload-queue"></div>
                </div>
                
                <div style="margin-top: 20px;">
                    <h3>Uploaded Files</h3>
                    <div id="uploaded-files"></div>
                </div>
            </div>
        </div>
    </div>

    <!-- Hidden file editor -->
    <div id="editor-modal" style="display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.8); z-index: 1000;">
        <div style="background: #111; width: 90%; height: 90%; margin: 2% auto; padding: 20px; border-radius: 10px;">
            <div style="display: flex; justify-content: space-between; margin-bottom: 15px;">
                <h3 id="editor-title">Editing: </h3>
                <button class="btn-danger" onclick="closeEditor()">Close</button>
            </div>
            <textarea id="editor-content" style="width: 100%; height: 80%; background: #000; color: #fff; padding: 15px; font-family: monospace;"></textarea>
            <div style="margin-top: 15px; text-align: right;">
                <button class="btn-success" onclick="saveFile()">💾 Save</button>
                <button class="btn" onclick="runCurrentFile()">▶ Run</button>
                <button class="btn-danger" onclick="deleteCurrentFile()">🗑 Delete</button>
            </div>
        </div>
    </div>

    <script>
        let currentTab = 'terminal';
        let commandHistory = [];
        let historyIndex = -1;
        let currentFilePath = '';
        
        // Tab switching
        function switchTab(tabName) {
            currentTab = tabName;
            document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
            document.querySelectorAll('.tab-content').forEach(t => t.style.display = 'none');
            
            event.target.classList.add('active');
            document.getElementById(tabName + '-tab').style.display = 'block';
            
            if (tabName === 'files') refreshFileList();
            if (tabName === 'processes') refreshProcesses();
            if (tabName === 'logs') refreshLogs();
            if (tabName === 'upload') loadUploadedFiles();
        }
        
        // Terminal functions
        function addTerminalLine(text, type = 'output') {
            const container = document.getElementById('terminal-output');
            const line = document.createElement('div');
            line.className = 'terminal-line ' + type;
            line.innerHTML = text;
            container.appendChild(line);
            container.scrollTop = container.scrollHeight;
        }
        
        function clearTerminal() {
            document.getElementById('terminal-output').innerHTML = '';
            addTerminalLine('Terminal cleared', 'success');
        }
        
        function executeCommand() {
            const input = document.getElementById('command-input');
            const cmd = input.value.trim();
            if (!cmd) return;
            
            commandHistory.push(cmd);
            historyIndex = -1;
            
            // Display command
            addTerminalLine('<span class="prompt">$</span> <span class="command">' + cmd + '</span>');
            input.value = '';
            
            // Handle special commands
            if (cmd === 'clear') {
                clearTerminal();
                return;
            }
            
            if (cmd === 'help') {
                showHelp();
                return;
            }
            
            if (cmd.startsWith('/stop ')) {
                const filename = cmd.substring(6).trim();
                stopScript(filename);
                return;
            }
            
            // Execute via API
            fetch('/api/execute', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({command: cmd})
            })
            .then(response => {
                const reader = response.body.getReader();
                const decoder = new TextDecoder();
                
                function read() {
                    reader.read().then(({done, value}) => {
                        if (done) {
                            updateStatus();
                            return;
                        }
                        
                        const text = decoder.decode(value);
                        if (text.trim()) {
                            addTerminalLine(text, 'output');
                        }
                        
                        read();
                    });
                }
                
                read();
            })
            .catch(error => {
                addTerminalLine('Error: ' + error.message, 'error');
            });
        }
        
        function runCmd(cmd) {
            document.getElementById('command-input').value = cmd;
            executeCommand();
        }
        
        function showHelp() {
            addTerminalLine('<span class="info">=== Available Commands ===</span>');
            addTerminalLine('<span class="output">• All Termux/Linux commands</span>');
            addTerminalLine('<span class="output">• python3 script.py - Run Python</span>');
            addTerminalLine('<span class="output">• /stop script.py - Stop script</span>');
            addTerminalLine('<span class="output">• nano file.txt - Edit file</span>');
            addTerminalLine('<span class="output">• pip install package</span>');
            addTerminalLine('<span class="output">• apt update/upgrade</span>');
            addTerminalLine('<span class="output">• git clone/commit/push</span>');
            addTerminalLine('<span class="output">• ssh/scp/wget/curl</span>');
        }
        
        // Process management
        function stopScript(filename) {
            fetch('/api/stop', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({filename: filename})
            })
            .then(r => r.json())
            .then(data => {
                if (data.success) {
                    addTerminalLine('✅ Stopped: ' + filename, 'success');
                } else {
                    addTerminalLine('❌ Error: ' + data.message, 'error');
                }
                refreshProcesses();
            });
        }
        
        function killAllProcesses() {
            if (!confirm('Kill all running processes?')) return;
            
            fetch('/api/kill_all', {method: 'POST'})
            .then(r => r.json())
            .then(data => {
                if (data.success) {
                    addTerminalLine('✅ Killed all processes', 'success');
                } else {
                    addTerminalLine('❌ Error: ' + data.message, 'error');
                }
                refreshProcesses();
            });
        }
        
        function sendCtrl(key) {
            fetch('/api/ctrl', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({key: key})
            })
            .then(r => r.json())
            .then(data => {
                addTerminalLine('Sent Ctrl+' + key, 'info');
            });
        }
        
        // File management
        function refreshFileList() {
            fetch('/api/files')
            .then(r => r.json())
            .then(data => {
                let html = '';
                data.files.forEach(file => {
                    const icon = file.type === 'dir' ? '📁' : 
                                file.name.endsWith('.py') ? '🐍' :
                                file.name.endsWith('.txt') ? '📝' :
                                file.name.endsWith('.sh') ? '💻' : '📄';
                    
                    html += `
                        <div class="file-item ${file.type}" onclick="handleFileClick('${file.name}', ${file.type === 'dir'})" style="display: flex; justify-content: space-between;">
                            <div>${icon} ${file.name}</div>
                            <div style="color: #888;">${file.size}</div>
                        </div>
                    `;
                });
                document.getElementById('file-list').innerHTML = html;
            });
        }
        
        function handleFileClick(name, isDir) {
            if (isDir) {
                // Change directory
                fetch('/api/cd', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({dir: name})
                })
                .then(r => r.json())
                .then(data => {
                    if (data.success) {
                        document.getElementById('file-path').value = data.current_dir;
                        refreshFileList();
                    }
                });
            } else {
                // Open file in editor
                openEditor(name);
            }
        }
        
        function openEditor(filename) {
            fetch('/api/read_file?file=' + encodeURIComponent(filename))
            .then(r => r.text())
            .then(content => {
                document.getElementById('editor-title').textContent = 'Editing: ' + filename;
                document.getElementById('editor-content').value = content;
                currentFilePath = filename;
                document.getElementById('editor-modal').style.display = 'block';
            });
        }
        
        function closeEditor() {
            document.getElementById('editor-modal').style.display = 'none';
            currentFilePath = '';
        }
        
        function saveFile() {
            const content = document.getElementById('editor-content').value;
            fetch('/api/write_file', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({
                    file: currentFilePath,
                    content: content
                })
            })
            .then(r => r.json())
            .then(data => {
                if (data.success) {
                    alert('File saved successfully!');
                } else {
                    alert('Error saving file: ' + data.message);
                }
            });
        }
        
        function runCurrentFile() {
            if (currentFilePath.endsWith('.py')) {
                document.getElementById('command-input').value = 'python3 ' + currentFilePath;
                executeCommand();
                closeEditor();
            } else if (currentFilePath.endsWith('.sh')) {
                document.getElementById('command-input').value = 'bash ' + currentFilePath;
                executeCommand();
                closeEditor();
            } else {
                alert('Cannot run this file type');
            }
        }
        
        function deleteCurrentFile() {
            if (!confirm('Delete ' + currentFilePath + '?')) return;
            
            fetch('/api/delete', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({file: currentFilePath})
            })
            .then(r => r.json())
            .then(data => {
                if (data.success) {
                    alert('File deleted');
                    closeEditor();
                    refreshFileList();
                } else {
                    alert('Error: ' + data.message);
                }
            });
        }
        
        function changeFileDir() {
            const path = document.getElementById('file-path').value;
            fetch('/api/cd', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({dir: path})
            })
            .then(r => r.json())
            .then(data => {
                if (data.success) {
                    document.getElementById('file-path').value = data.current_dir;
                    refreshFileList();
                }
            });
        }
        
        // Upload functions
        function uploadFile() {
            switchTab('upload');
        }
        
        function handleFileSelect(event) {
            const files = event.target.files;
            for (let file of files) {
                uploadSingleFile(file);
            }
        }
        
        function uploadSingleFile(file) {
            const formData = new FormData();
            formData.append('file', file);
            
            const queue = document.getElementById('upload-queue');
            const item = document.createElement('div');
            item.innerHTML = `Uploading: ${file.name} (${Math.round(file.size/1024)}KB)...`;
            queue.appendChild(item);
            
            fetch('/api/upload', {
                method: 'POST',
                body: formData
            })
            .then(r => r.json())
            .then(data => {
                if (data.success) {
                    item.innerHTML = `✅ ${file.name} uploaded`;
                    loadUploadedFiles();
                } else {
                    item.innerHTML = `❌ ${file.name} failed: ${data.message}`;
                }
            });
        }
        
        function loadUploadedFiles() {
            fetch('/api/uploads')
            .then(r => r.json())
            .then(data => {
                let html = '';
                data.files.forEach(file => {
                    html += `
                        <div style="margin: 5px; padding: 5px; background: #222; border-radius: 3px;">
                            ${file.name} (${file.size})
                            <button onclick="runUploadedFile('${file.name}')" class="btn" style="padding: 2px 5px; margin-left: 10px;">Run</button>
                            <button onclick="deleteUploadedFile('${file.name}')" class="btn-danger" style="padding: 2px 5px; margin-left: 5px;">Delete</button>
                        </div>
                    `;
                });
                document.getElementById('uploaded-files').innerHTML = html;
            });
        }
        
        // Process monitoring
        function refreshProcesses() {
            fetch('/api/processes')
            .then(r => r.json())
            .then(data => {
                let html = '';
                if (data.processes.length === 0) {
                    html = '<div>No processes running</div>';
                } else {
                    data.processes.forEach(proc => {
                        html += `
                            <div class="process-item ${proc.status === 'running' ? '' : 'stopped'}">
                                <div><strong>PID ${proc.pid}</strong> - ${proc.cmd}</div>
                                <div>Type: ${proc.type} | Started: ${proc.start_time}</div>
                                <div>Status: <span style="color: ${proc.status === 'running' ? '#0f0' : '#f00'}">${proc.status}</span></div>
                                <button onclick="stopProcess(${proc.pid})" class="btn-danger" style="padding: 2px 5px; margin-top: 5px;">Stop</button>
                            </div>
                        `;
                    });
                }
                document.getElementById('process-list').innerHTML = html;
                document.getElementById('process-count').textContent = '🔄 ' + data.processes.length + ' processes';
            });
        }
        
        function stopProcess(pid) {
            fetch('/api/stop_pid', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({pid: pid})
            })
            .then(r => r.json())
            .then(data => {
                if (data.success) {
                    refreshProcesses();
                }
            });
        }
        
        // Logs
        function refreshLogs() {
            fetch('/api/logs')
            .then(r => r.text())
            .then(text => {
                const lines = text.split('\\n').reverse();
                let html = '';
                lines.forEach(line => {
                    if (line.trim()) {
                        html += `<div class="log-entry">${line}</div>`;
                    }
                });
                document.getElementById('logs-container').innerHTML = html;
            });
        }
        
        // Status updates
        function updateStatus() {
            fetch('/api/status')
            .then(r => r.json())
            .then(data => {
                document.getElementById('current-dir').textContent = '📁 ' + data.current_dir;
            });
            refreshProcesses();
        }
        
        // Keyboard shortcuts
        document.getElementById('command-input').addEventListener('keydown', function(e) {
            if (e.key === 'Enter') executeCommand();
            else if (e.key === 'ArrowUp') {
                e.preventDefault();
                if (commandHistory.length > 0) {
                    historyIndex = Math.min(historyIndex + 1, commandHistory.length - 1);
                    this.value = commandHistory[commandHistory.length - 1 - historyIndex];
                }
            } else if (e.key === 'ArrowDown') {
                e.preventDefault();
                if (historyIndex > 0) {
                    historyIndex--;
                    this.value = commandHistory[commandHistory.length - 1 - historyIndex];
                } else {
                    historyIndex = -1;
                    this.value = '';
                }
            } else if (e.ctrlKey && e.key === 'l') {
                e.preventDefault();
                clearTerminal();
            }
        });
        
        // Drag and drop for upload
        const dropArea = document.getElementById('drop-area');
        dropArea.addEventListener('dragover', (e) => {
            e.preventDefault();
            dropArea.style.background = '#222';
        });
        
        dropArea.addEventListener('dragleave', () => {
            dropArea.style.background = '';
        });
        
        dropArea.addEventListener('drop', (e) => {
            e.preventDefault();
            dropArea.style.background = '';
            const files = e.dataTransfer.files;
            for (let file of files) {
                uploadSingleFile(file);
            }
        });
        
        // Initialize
        document.addEventListener('DOMContentLoaded', function() {
            updateStatus();
            refreshFileList();
            refreshProcesses();
            setInterval(updateStatus, 5000);
            
            // Auto-focus input
            document.getElementById('command-input').focus();
        });
    </script>
</body>
</html>
'''
# ==================== FLASK ROUTES ====================
@app.route('/')
def index():
    return INDEX_HTML

@app.route('/api/execute', methods=['POST'])
def api_execute():
    data = request.json
    cmd = data.get('command', '').strip()
    if not cmd:
        return jsonify({'error': 'No command provided'})
    
    def generate():
        for line in execute_command_stream(cmd, state.current_dir):
            yield line
    
    return Response(generate(), mimetype='text/plain')

@app.route('/api/stop', methods=['POST'])
def api_stop():
    data = request.json
    filename = data.get('filename', '')
    log_message(f"Stop requested for: {filename}", "PROCESS")
    killed = []
    for pid, info in list(state.processes.items()):
        if filename in info['cmd']:
            if kill_process_tree(pid):
                del state.processes[pid]
                killed.append(pid)
    if killed:
        return jsonify({'success': True, 'message': f'Stopped {len(killed)} processes', 'killed': killed})
    return jsonify({'success': False, 'message': f'No process found for {filename}'})

@app.route('/api/stop_pid', methods=['POST'])
def api_stop_pid():
    data = request.json
    pid = data.get('pid')
    try:
        if kill_process_tree(pid):
            if pid in state.processes:
                del state.processes[pid]
            return jsonify({'success': True, 'message': f'Process {pid} stopped'})
        else:
            return jsonify({'success': False, 'message': f'Process {pid} not found'})
    except Exception as e:
        return jsonify({'success': False, 'message': str(e)})

@app.route('/api/kill_all', methods=['POST'])
def api_kill_all():
    killed = []
    for pid in list(state.processes.keys()):
        if kill_process_tree(pid):
            killed.append(pid)
            del state.processes[pid]
    log_message(f"Killed all processes: {killed}", "SYSTEM")
    return jsonify({'success': True, 'message': f'Killed {len(killed)} processes', 'killed': killed})

@app.route('/api/status')
def api_status():
    return jsonify({
        'current_dir': state.current_dir,
        'process_count': len(state.processes),
        'logs_count': len(state.logs)
    })

# ==================== MAIN ====================
def open_browser():
    time.sleep(1)
    try:
        webbrowser.open('http://localhost:5000')
    except:
        pass

if __name__ == '__main__':
    print("\n" + "="*60)
    print("🚀 Termux Web Terminal Pro v1.0")
    print("="*60)
    print(f"📁 Current Directory: {os.getcwd()}")
    print(f"🌐 Web Interface: http://localhost:5000")
    print(f"📂 Upload Directory: {state.upload_dir}")
    print("="*60)
    print("\nStarting server...")

    threading.Thread(target=open_browser, daemon=True).start()
    app.run(host='0.0.0.0', port=5000, debug=False, threaded=True)
