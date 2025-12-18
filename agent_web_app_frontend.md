# Codex Multi-Agent Web Application - Frontend Implementation Plan

## Overview

Build a modern, production-ready React frontend for the Codex Multi-Agent System. The UI provides project management, real-time chat with individual agents, live terminal streaming, and conversation history grouped by projects.

**Tech Stack:**
- React 18 with TypeScript
- Vite for build tooling
- TailwindCSS for styling
- Zustand for state management
- React Query for server state
- Xterm.js for terminal emulation
- React Router for navigation

---

## UI Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Navigation Bar                                  │
│  [Logo]  [Projects ▼]  [New Project]                    [Settings] [User]   │
├────────────────────┬────────────────────────────────────────────────────────┤
│                    │                                                         │
│   Project Sidebar  │                    Main Content Area                    │
│                    │                                                         │
│   ┌──────────────┐ │  ┌─────────────────────────────────────────────────┐   │
│   │ Project A    │ │  │                                                 │   │
│   │  └─ Session 1│ │  │              Chat / Terminal View               │   │
│   │  └─ Session 2│ │  │                                                 │   │
│   ├──────────────┤ │  │                                                 │   │
│   │ Project B    │ │  │                                                 │   │
│   │  └─ Session 1│ │  │                                                 │   │
│   ├──────────────┤ │  └─────────────────────────────────────────────────┘   │
│   │ + New Project│ │                                                         │
│   └──────────────┘ │  ┌─────────────────────────────────────────────────┐   │
│                    │  │              Agent Selector / Status             │   │
│   Agent Quick      │  │  [Planner] [Builder] [QA/QC] [ProdReady]        │   │
│   Access           │  └─────────────────────────────────────────────────┘   │
│   ┌──────────────┐ │                                                         │
│   │ 🎯 Planner   │ │  ┌─────────────────────────────────────────────────┐   │
│   │ 🔨 Builder   │ │  │              Input Area                         │   │
│   │ ✅ QA/QC     │ │  │  [Message input...            ] [Send] [Upload] │   │
│   │ 🚀 ProdReady │ │  └─────────────────────────────────────────────────┘   │
│   └──────────────┘ │                                                         │
│                    │                                                         │
└────────────────────┴────────────────────────────────────────────────────────┘
```

---

## Phase 1: Project Setup

### 1.1 Initialize Vite React Project

```bash
# Create project
npm create vite@latest frontend -- --template react-ts
cd frontend

# Install dependencies
npm install react-router-dom @tanstack/react-query zustand
npm install tailwindcss postcss autoprefixer
npm install @xterm/xterm @xterm/addon-fit @xterm/addon-web-links
npm install lucide-react clsx date-fns
npm install axios
npm install -D @types/node

# Initialize Tailwind
npx tailwindcss init -p
```

### 1.2 Tailwind Configuration

**File: `tailwind.config.js`**

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        // Brand colors
        primary: {
          50: '#f0f9ff',
          100: '#e0f2fe',
          200: '#bae6fd',
          300: '#7dd3fc',
          400: '#38bdf8',
          500: '#0ea5e9',
          600: '#0284c7',
          700: '#0369a1',
          800: '#075985',
          900: '#0c4a6e',
          950: '#082f49',
        },
        // Agent colors
        agent: {
          planner: '#8b5cf6',    // Purple
          builder: '#f59e0b',    // Amber
          qa: '#10b981',         // Emerald
          prod: '#ef4444',       // Red
        },
        // Terminal colors
        terminal: {
          bg: '#1a1b26',
          text: '#a9b1d6',
          green: '#9ece6a',
          red: '#f7768e',
          yellow: '#e0af68',
          blue: '#7aa2f7',
        }
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'Fira Code', 'monospace'],
      },
      animation: {
        'pulse-slow': 'pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite',
        'slide-up': 'slideUp 0.3s ease-out',
        'slide-down': 'slideDown 0.3s ease-out',
        'fade-in': 'fadeIn 0.2s ease-out',
      },
      keyframes: {
        slideUp: {
          '0%': { transform: 'translateY(10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
        slideDown: {
          '0%': { transform: 'translateY(-10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
      },
    },
  },
  plugins: [],
}
```

### 1.3 Global Styles

**File: `src/index.css`**

```css
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=JetBrains+Mono:wght@400;500;600&display=swap');

@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  html {
    @apply antialiased;
  }

  body {
    @apply bg-gray-50 text-gray-900 dark:bg-gray-950 dark:text-gray-100;
  }

  /* Custom scrollbar */
  ::-webkit-scrollbar {
    @apply w-2 h-2;
  }

  ::-webkit-scrollbar-track {
    @apply bg-gray-100 dark:bg-gray-800;
  }

  ::-webkit-scrollbar-thumb {
    @apply bg-gray-300 dark:bg-gray-600 rounded-full;
  }

  ::-webkit-scrollbar-thumb:hover {
    @apply bg-gray-400 dark:bg-gray-500;
  }
}

@layer components {
  .btn {
    @apply inline-flex items-center justify-center px-4 py-2
           font-medium rounded-lg transition-all duration-200
           focus:outline-none focus:ring-2 focus:ring-offset-2
           disabled:opacity-50 disabled:cursor-not-allowed;
  }

  .btn-primary {
    @apply btn bg-primary-600 text-white hover:bg-primary-700
           focus:ring-primary-500 dark:bg-primary-500 dark:hover:bg-primary-600;
  }

  .btn-secondary {
    @apply btn bg-gray-100 text-gray-700 hover:bg-gray-200
           focus:ring-gray-500 dark:bg-gray-800 dark:text-gray-200 dark:hover:bg-gray-700;
  }

  .btn-ghost {
    @apply btn bg-transparent hover:bg-gray-100 dark:hover:bg-gray-800;
  }

  .card {
    @apply bg-white dark:bg-gray-900 rounded-xl shadow-sm
           border border-gray-200 dark:border-gray-800;
  }

  .input {
    @apply w-full px-4 py-2 rounded-lg border border-gray-300
           dark:border-gray-700 bg-white dark:bg-gray-800
           focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-transparent
           placeholder-gray-400 dark:placeholder-gray-500;
  }

  .agent-badge {
    @apply inline-flex items-center px-2.5 py-0.5 rounded-full
           text-xs font-medium;
  }

  .agent-badge-planner {
    @apply agent-badge bg-purple-100 text-purple-800
           dark:bg-purple-900/30 dark:text-purple-300;
  }

  .agent-badge-builder {
    @apply agent-badge bg-amber-100 text-amber-800
           dark:bg-amber-900/30 dark:text-amber-300;
  }

  .agent-badge-qa {
    @apply agent-badge bg-emerald-100 text-emerald-800
           dark:bg-emerald-900/30 dark:text-emerald-300;
  }

  .agent-badge-prod {
    @apply agent-badge bg-red-100 text-red-800
           dark:bg-red-900/30 dark:text-red-300;
  }
}
```

**Checklist:**
- [ ] Initialize Vite project with React + TypeScript
- [ ] Install all dependencies
- [ ] Configure Tailwind CSS
- [ ] Set up global styles
- [ ] Configure path aliases in vite.config.ts

---

## Phase 2: Core Infrastructure

### 2.1 API Client

**File: `src/lib/api.ts`**

```typescript
import axios, { AxiosInstance, AxiosError } from 'axios';

const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000/api';

export const api: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
  timeout: 30000,
});

// Request interceptor
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('auth_token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor
api.interceptors.response.use(
  (response) => response,
  (error: AxiosError) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('auth_token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

// ============== TYPE DEFINITIONS ==============

export interface Project {
  id: string;
  name: string;
  description: string | null;
  workspace_path: string;
  created_at: string;
  updated_at: string;
  is_active: boolean;
  settings: Record<string, unknown>;
}

export interface Session {
  id: string;
  project_id: string;
  title: string | null;
  user_request: string;
  status: 'pending' | 'running' | 'completed' | 'failed' | 'cancelled';
  current_agent: AgentRole | null;
  created_at: string;
  completed_at: string | null;
  agents_md_content: string | null;
}

export interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system' | 'tool';
  content: string;
  created_at: string;
}

export interface Conversation {
  id: string;
  agent_type: AgentRole;
  status: 'pending' | 'running' | 'completed' | 'failed';
  started_at: string;
  ended_at: string | null;
  messages: Message[];
}

export interface AgentInfo {
  role: AgentRole;
  name: string;
  description: string;
  handoff_description: string;
}

export interface ProjectFile {
  id: string;
  filename: string;
  file_type: string;
  file_size: number;
  uploaded_at: string;
}

export type AgentRole = 'planner' | 'builder' | 'qa_qc' | 'prod_ready';

// ============== API FUNCTIONS ==============

// Projects
export const projectsApi = {
  list: () => api.get<{ projects: Project[]; total: number }>('/projects'),
  get: (id: string) => api.get<Project>(`/projects/${id}`),
  create: (data: { name: string; description?: string }) =>
    api.post<Project>('/projects', data),
  delete: (id: string) => api.delete(`/projects/${id}`),
};

// Sessions
export const sessionsApi = {
  create: (data: { project_id: string; user_request: string; title?: string }) =>
    api.post<Session>('/sessions', data),
  get: (id: string) => api.get<Session>(`/sessions/${id}`),
  start: (id: string) => api.post(`/sessions/${id}/start`),
  getConversations: (id: string) =>
    api.get<Conversation[]>(`/sessions/${id}/conversations`),
  listByProject: (projectId: string) =>
    api.get<Session[]>(`/sessions/project/${projectId}`),
};

// Agents
export const agentsApi = {
  list: () => api.get<AgentInfo[]>('/agents'),
  get: (role: AgentRole) => api.get<AgentInfo>(`/agents/${role}`),
  chat: (data: {
    project_id: string;
    session_id: string;
    agent_role: AgentRole;
    message: string
  }) => api.post<{ agent_role: AgentRole; response: string }>('/agents/chat', data),
};

// Files
export const filesApi = {
  upload: (projectId: string, file: File) => {
    const formData = new FormData();
    formData.append('file', file);
    return api.post<ProjectFile>(`/files/upload/${projectId}`, formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
    });
  },
  list: (projectId: string) => api.get<ProjectFile[]>(`/files/project/${projectId}`),
  delete: (fileId: string) => api.delete(`/files/${fileId}`),
};
```

### 2.2 WebSocket Client

**File: `src/lib/websocket.ts`**

```typescript
type MessageHandler = (data: WebSocketMessage) => void;

export interface WebSocketMessage {
  type:
    | 'workflow_progress'
    | 'workflow_complete'
    | 'agent_message'
    | 'codex_output'
    | 'error'
    | 'pong';
  stage?: string;
  message?: string;
  result?: Record<string, unknown>;
  agent_role?: string;
  content?: string;
  task_id?: string;
  line?: string;
}

export class WebSocketClient {
  private ws: WebSocket | null = null;
  private url: string;
  private handlers: Set<MessageHandler> = new Set();
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;
  private pingInterval: number | null = null;

  constructor(sessionId: string) {
    const wsBase = import.meta.env.VITE_WS_URL || 'ws://localhost:8000/ws';
    this.url = `${wsBase}/session/${sessionId}`;
  }

  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      try {
        this.ws = new WebSocket(this.url);

        this.ws.onopen = () => {
          console.log('WebSocket connected');
          this.reconnectAttempts = 0;
          this.startPing();
          resolve();
        };

        this.ws.onmessage = (event) => {
          try {
            const data = JSON.parse(event.data) as WebSocketMessage;
            this.handlers.forEach(handler => handler(data));
          } catch (e) {
            console.error('Failed to parse WebSocket message:', e);
          }
        };

        this.ws.onclose = () => {
          console.log('WebSocket disconnected');
          this.stopPing();
          this.attemptReconnect();
        };

        this.ws.onerror = (error) => {
          console.error('WebSocket error:', error);
          reject(error);
        };
      } catch (error) {
        reject(error);
      }
    });
  }

  private startPing() {
    this.pingInterval = window.setInterval(() => {
      this.send({ action: 'ping' });
    }, 30000);
  }

  private stopPing() {
    if (this.pingInterval) {
      clearInterval(this.pingInterval);
      this.pingInterval = null;
    }
  }

  private attemptReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
      console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);
      setTimeout(() => this.connect(), delay);
    }
  }

  subscribe(handler: MessageHandler): () => void {
    this.handlers.add(handler);
    return () => this.handlers.delete(handler);
  }

  send(data: Record<string, unknown>) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    }
  }

  // Start the 4-agent workflow
  startWorkflow(projectId: string, userRequest: string) {
    this.send({
      action: 'start_workflow',
      project_id: projectId,
      user_request: userRequest,
    });
  }

  // Chat with a specific agent
  chatWithAgent(projectId: string, agentRole: string, message: string) {
    this.send({
      action: 'chat',
      project_id: projectId,
      agent_role: agentRole,
      message: message,
    });
  }

  // Subscribe to codex terminal output
  subscribeToCodex(taskId: string) {
    this.send({
      action: 'subscribe_codex',
      task_id: taskId,
    });
  }

  disconnect() {
    this.stopPing();
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }
}

// Terminal WebSocket for live codex output
export class TerminalWebSocket {
  private ws: WebSocket | null = null;
  private onOutput: (line: string) => void;
  private onComplete: () => void;

  constructor(
    taskId: string,
    onOutput: (line: string) => void,
    onComplete: () => void
  ) {
    const wsBase = import.meta.env.VITE_WS_URL || 'ws://localhost:8000/ws';
    const url = `${wsBase}/terminal/${taskId}`;

    this.onOutput = onOutput;
    this.onComplete = onComplete;

    this.ws = new WebSocket(url);

    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      if (data.type === 'output') {
        this.onOutput(data.line);
      } else if (data.type === 'complete') {
        this.onComplete();
      }
    };
  }

  disconnect() {
    this.ws?.close();
  }
}
```

### 2.3 Global State Store

**File: `src/stores/appStore.ts`**

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { Project, Session, AgentRole, AgentInfo, Conversation } from '@/lib/api';

interface AppState {
  // Theme
  theme: 'light' | 'dark' | 'system';
  setTheme: (theme: 'light' | 'dark' | 'system') => void;

  // Current selections
  currentProject: Project | null;
  setCurrentProject: (project: Project | null) => void;

  currentSession: Session | null;
  setCurrentSession: (session: Session | null) => void;

  selectedAgent: AgentRole | null;
  setSelectedAgent: (agent: AgentRole | null) => void;

  // UI state
  sidebarOpen: boolean;
  toggleSidebar: () => void;

  terminalVisible: boolean;
  setTerminalVisible: (visible: boolean) => void;

  // Agents info (cached)
  agents: AgentInfo[];
  setAgents: (agents: AgentInfo[]) => void;

  // Active conversations in current session
  conversations: Conversation[];
  setConversations: (conversations: Conversation[]) => void;
  addMessage: (conversationId: string, message: { role: string; content: string }) => void;

  // Workflow state
  workflowStatus: 'idle' | 'running' | 'completed' | 'failed';
  workflowStage: string | null;
  setWorkflowStatus: (status: 'idle' | 'running' | 'completed' | 'failed', stage?: string) => void;

  // Terminal output
  terminalOutput: string[];
  addTerminalOutput: (line: string) => void;
  clearTerminalOutput: () => void;
}

export const useAppStore = create<AppState>()(
  persist(
    (set) => ({
      // Theme
      theme: 'system',
      setTheme: (theme) => set({ theme }),

      // Current selections
      currentProject: null,
      setCurrentProject: (project) => set({ currentProject: project }),

      currentSession: null,
      setCurrentSession: (session) => set({ currentSession: session }),

      selectedAgent: null,
      setSelectedAgent: (agent) => set({ selectedAgent: agent }),

      // UI state
      sidebarOpen: true,
      toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),

      terminalVisible: false,
      setTerminalVisible: (visible) => set({ terminalVisible: visible }),

      // Agents
      agents: [],
      setAgents: (agents) => set({ agents }),

      // Conversations
      conversations: [],
      setConversations: (conversations) => set({ conversations }),
      addMessage: (conversationId, message) => set((state) => ({
        conversations: state.conversations.map(conv =>
          conv.id === conversationId
            ? {
                ...conv,
                messages: [...conv.messages, {
                  id: crypto.randomUUID(),
                  ...message,
                  created_at: new Date().toISOString()
                } as any]
              }
            : conv
        )
      })),

      // Workflow
      workflowStatus: 'idle',
      workflowStage: null,
      setWorkflowStatus: (status, stage) => set({
        workflowStatus: status,
        workflowStage: stage ?? null
      }),

      // Terminal
      terminalOutput: [],
      addTerminalOutput: (line) => set((state) => ({
        terminalOutput: [...state.terminalOutput, line].slice(-1000) // Keep last 1000 lines
      })),
      clearTerminalOutput: () => set({ terminalOutput: [] }),
    }),
    {
      name: 'codex-agents-storage',
      partialize: (state) => ({
        theme: state.theme,
        sidebarOpen: state.sidebarOpen,
      }),
    }
  )
);
```

### 2.4 React Query Setup

**File: `src/lib/queryClient.ts`**

```typescript
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60, // 1 minute
      retry: 2,
      refetchOnWindowFocus: false,
    },
  },
});
```

**File: `src/hooks/useProjects.ts`**

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { projectsApi, type Project } from '@/lib/api';

export function useProjects() {
  return useQuery({
    queryKey: ['projects'],
    queryFn: async () => {
      const response = await projectsApi.list();
      return response.data.projects;
    },
  });
}

export function useProject(id: string | null) {
  return useQuery({
    queryKey: ['project', id],
    queryFn: async () => {
      if (!id) return null;
      const response = await projectsApi.get(id);
      return response.data;
    },
    enabled: !!id,
  });
}

export function useCreateProject() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: { name: string; description?: string }) => {
      const response = await projectsApi.create(data);
      return response.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
}

export function useDeleteProject() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (id: string) => {
      await projectsApi.delete(id);
      return id;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
}
```

**File: `src/hooks/useSessions.ts`**

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { sessionsApi, type Session } from '@/lib/api';

export function useProjectSessions(projectId: string | null) {
  return useQuery({
    queryKey: ['sessions', projectId],
    queryFn: async () => {
      if (!projectId) return [];
      const response = await sessionsApi.listByProject(projectId);
      return response.data;
    },
    enabled: !!projectId,
  });
}

export function useSession(id: string | null) {
  return useQuery({
    queryKey: ['session', id],
    queryFn: async () => {
      if (!id) return null;
      const response = await sessionsApi.get(id);
      return response.data;
    },
    enabled: !!id,
  });
}

export function useSessionConversations(sessionId: string | null) {
  return useQuery({
    queryKey: ['conversations', sessionId],
    queryFn: async () => {
      if (!sessionId) return [];
      const response = await sessionsApi.getConversations(sessionId);
      return response.data;
    },
    enabled: !!sessionId,
  });
}

export function useCreateSession() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: {
      project_id: string;
      user_request: string;
      title?: string
    }) => {
      const response = await sessionsApi.create(data);
      return response.data;
    },
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['sessions', data.project_id] });
    },
  });
}

export function useStartSession() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (sessionId: string) => {
      const response = await sessionsApi.start(sessionId);
      return response.data;
    },
    onSuccess: (_, sessionId) => {
      queryClient.invalidateQueries({ queryKey: ['session', sessionId] });
    },
  });
}
```

**Checklist:**
- [ ] Create API client with all endpoints
- [ ] Implement WebSocket client
- [ ] Set up Zustand store
- [ ] Configure React Query
- [ ] Create all data hooks

---

## Phase 3: Layout Components

### 3.1 Root Layout

**File: `src/components/layout/RootLayout.tsx`**

```typescript
import { Outlet } from 'react-router-dom';
import { useEffect } from 'react';
import { useAppStore } from '@/stores/appStore';
import { Navbar } from './Navbar';
import { Sidebar } from './Sidebar';
import { TerminalPanel } from '../terminal/TerminalPanel';

export function RootLayout() {
  const { theme, sidebarOpen, terminalVisible } = useAppStore();

  // Handle theme
  useEffect(() => {
    const root = document.documentElement;

    if (theme === 'system') {
      const systemDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      root.classList.toggle('dark', systemDark);
    } else {
      root.classList.toggle('dark', theme === 'dark');
    }
  }, [theme]);

  return (
    <div className="h-screen flex flex-col bg-gray-50 dark:bg-gray-950">
      {/* Top Navigation */}
      <Navbar />

      {/* Main Content */}
      <div className="flex-1 flex overflow-hidden">
        {/* Sidebar */}
        <aside
          className={`
            ${sidebarOpen ? 'w-72' : 'w-0'}
            transition-all duration-300 ease-in-out
            border-r border-gray-200 dark:border-gray-800
            bg-white dark:bg-gray-900
            overflow-hidden
          `}
        >
          <Sidebar />
        </aside>

        {/* Main Area */}
        <main className="flex-1 flex flex-col overflow-hidden">
          {/* Page Content */}
          <div className="flex-1 overflow-auto">
            <Outlet />
          </div>

          {/* Terminal Panel */}
          {terminalVisible && (
            <div className="h-64 border-t border-gray-200 dark:border-gray-800">
              <TerminalPanel />
            </div>
          )}
        </main>
      </div>
    </div>
  );
}
```

### 3.2 Navigation Bar

**File: `src/components/layout/Navbar.tsx`**

```typescript
import {
  Menu,
  Sun,
  Moon,
  Terminal,
  Settings,
  PanelLeftClose,
  PanelLeft
} from 'lucide-react';
import { useAppStore } from '@/stores/appStore';

export function Navbar() {
  const {
    theme,
    setTheme,
    sidebarOpen,
    toggleSidebar,
    terminalVisible,
    setTerminalVisible,
    currentProject,
    workflowStatus,
    workflowStage
  } = useAppStore();

  const toggleTheme = () => {
    setTheme(theme === 'dark' ? 'light' : 'dark');
  };

  return (
    <nav className="h-14 border-b border-gray-200 dark:border-gray-800 bg-white dark:bg-gray-900 px-4 flex items-center justify-between">
      {/* Left Section */}
      <div className="flex items-center gap-4">
        <button
          onClick={toggleSidebar}
          className="p-2 hover:bg-gray-100 dark:hover:bg-gray-800 rounded-lg transition-colors"
          aria-label="Toggle sidebar"
        >
          {sidebarOpen ? (
            <PanelLeftClose className="w-5 h-5" />
          ) : (
            <PanelLeft className="w-5 h-5" />
          )}
        </button>

        <div className="flex items-center gap-2">
          <div className="w-8 h-8 bg-gradient-to-br from-primary-500 to-primary-700 rounded-lg flex items-center justify-center">
            <span className="text-white font-bold text-sm">CA</span>
          </div>
          <span className="font-semibold text-lg">Codex Agents</span>
        </div>

        {/* Current Project Indicator */}
        {currentProject && (
          <div className="ml-4 px-3 py-1 bg-gray-100 dark:bg-gray-800 rounded-full text-sm">
            {currentProject.name}
          </div>
        )}

        {/* Workflow Status */}
        {workflowStatus === 'running' && (
          <div className="ml-4 flex items-center gap-2 text-sm text-primary-600 dark:text-primary-400">
            <div className="w-2 h-2 bg-primary-500 rounded-full animate-pulse" />
            <span>Running: {workflowStage}</span>
          </div>
        )}
      </div>

      {/* Right Section */}
      <div className="flex items-center gap-2">
        {/* Terminal Toggle */}
        <button
          onClick={() => setTerminalVisible(!terminalVisible)}
          className={`
            p-2 rounded-lg transition-colors
            ${terminalVisible
              ? 'bg-primary-100 text-primary-600 dark:bg-primary-900/30 dark:text-primary-400'
              : 'hover:bg-gray-100 dark:hover:bg-gray-800'
            }
          `}
          aria-label="Toggle terminal"
        >
          <Terminal className="w-5 h-5" />
        </button>

        {/* Theme Toggle */}
        <button
          onClick={toggleTheme}
          className="p-2 hover:bg-gray-100 dark:hover:bg-gray-800 rounded-lg transition-colors"
          aria-label="Toggle theme"
        >
          {theme === 'dark' ? (
            <Sun className="w-5 h-5" />
          ) : (
            <Moon className="w-5 h-5" />
          )}
        </button>

        {/* Settings */}
        <button
          className="p-2 hover:bg-gray-100 dark:hover:bg-gray-800 rounded-lg transition-colors"
          aria-label="Settings"
        >
          <Settings className="w-5 h-5" />
        </button>
      </div>
    </nav>
  );
}
```

### 3.3 Sidebar

**File: `src/components/layout/Sidebar.tsx`**

```typescript
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  FolderOpen,
  Plus,
  ChevronDown,
  ChevronRight,
  MessageSquare,
  Target,
  Hammer,
  CheckCircle,
  Rocket
} from 'lucide-react';
import { useAppStore } from '@/stores/appStore';
import { useProjects } from '@/hooks/useProjects';
import { useProjectSessions } from '@/hooks/useSessions';
import { formatDistanceToNow } from 'date-fns';
import type { AgentRole } from '@/lib/api';

const AGENT_CONFIG: Record<AgentRole, { icon: typeof Target; label: string; color: string }> = {
  planner: { icon: Target, label: 'Planner', color: 'text-purple-500' },
  builder: { icon: Hammer, label: 'Builder', color: 'text-amber-500' },
  qa_qc: { icon: CheckCircle, label: 'QA/QC', color: 'text-emerald-500' },
  prod_ready: { icon: Rocket, label: 'Production', color: 'text-red-500' },
};

export function Sidebar() {
  const navigate = useNavigate();
  const { currentProject, setCurrentProject, setSelectedAgent, selectedAgent } = useAppStore();
  const { data: projects, isLoading: projectsLoading } = useProjects();
  const { data: sessions } = useProjectSessions(currentProject?.id ?? null);

  const [expandedProjects, setExpandedProjects] = useState<Set<string>>(new Set());

  const toggleProject = (projectId: string) => {
    const newExpanded = new Set(expandedProjects);
    if (newExpanded.has(projectId)) {
      newExpanded.delete(projectId);
    } else {
      newExpanded.add(projectId);
    }
    setExpandedProjects(newExpanded);
  };

  return (
    <div className="h-full flex flex-col p-4">
      {/* Projects Section */}
      <div className="flex-1 overflow-auto">
        <div className="flex items-center justify-between mb-3">
          <h3 className="text-sm font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wide">
            Projects
          </h3>
          <button
            onClick={() => navigate('/new-project')}
            className="p-1 hover:bg-gray-100 dark:hover:bg-gray-800 rounded transition-colors"
            aria-label="New project"
          >
            <Plus className="w-4 h-4" />
          </button>
        </div>

        {projectsLoading ? (
          <div className="space-y-2">
            {[1, 2, 3].map(i => (
              <div key={i} className="h-10 bg-gray-100 dark:bg-gray-800 rounded-lg animate-pulse" />
            ))}
          </div>
        ) : (
          <div className="space-y-1">
            {projects?.map(project => (
              <div key={project.id}>
                <button
                  onClick={() => {
                    setCurrentProject(project);
                    toggleProject(project.id);
                  }}
                  className={`
                    w-full flex items-center gap-2 px-3 py-2 rounded-lg text-left
                    transition-colors
                    ${currentProject?.id === project.id
                      ? 'bg-primary-50 text-primary-700 dark:bg-primary-900/30 dark:text-primary-300'
                      : 'hover:bg-gray-100 dark:hover:bg-gray-800'
                    }
                  `}
                >
                  {expandedProjects.has(project.id) ? (
                    <ChevronDown className="w-4 h-4 flex-shrink-0" />
                  ) : (
                    <ChevronRight className="w-4 h-4 flex-shrink-0" />
                  )}
                  <FolderOpen className="w-4 h-4 flex-shrink-0" />
                  <span className="truncate flex-1">{project.name}</span>
                </button>

                {/* Sessions under project */}
                {expandedProjects.has(project.id) && currentProject?.id === project.id && sessions && (
                  <div className="ml-6 mt-1 space-y-1">
                    {sessions.map(session => (
                      <button
                        key={session.id}
                        onClick={() => navigate(`/session/${session.id}`)}
                        className="w-full flex items-center gap-2 px-3 py-1.5 rounded-lg text-sm text-left hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors"
                      >
                        <MessageSquare className="w-3 h-3 flex-shrink-0 text-gray-400" />
                        <span className="truncate flex-1">
                          {session.title || session.user_request.slice(0, 30)}...
                        </span>
                        <span className="text-xs text-gray-400">
                          {formatDistanceToNow(new Date(session.created_at), { addSuffix: true })}
                        </span>
                      </button>
                    ))}
                  </div>
                )}
              </div>
            ))}
          </div>
        )}
      </div>

      {/* Agents Quick Access */}
      <div className="mt-4 pt-4 border-t border-gray-200 dark:border-gray-800">
        <h3 className="text-sm font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wide mb-3">
          Chat with Agent
        </h3>
        <div className="grid grid-cols-2 gap-2">
          {(Object.entries(AGENT_CONFIG) as [AgentRole, typeof AGENT_CONFIG[AgentRole]][]).map(
            ([role, config]) => {
              const Icon = config.icon;
              return (
                <button
                  key={role}
                  onClick={() => {
                    setSelectedAgent(role);
                    if (currentProject) {
                      navigate(`/chat/${role}`);
                    }
                  }}
                  disabled={!currentProject}
                  className={`
                    flex items-center gap-2 px-3 py-2 rounded-lg text-sm
                    transition-colors disabled:opacity-50 disabled:cursor-not-allowed
                    ${selectedAgent === role
                      ? 'bg-gray-100 dark:bg-gray-800'
                      : 'hover:bg-gray-50 dark:hover:bg-gray-800/50'
                    }
                  `}
                >
                  <Icon className={`w-4 h-4 ${config.color}`} />
                  <span>{config.label}</span>
                </button>
              );
            }
          )}
        </div>
      </div>
    </div>
  );
}
```

**Checklist:**
- [ ] Implement RootLayout with responsive design
- [ ] Create Navbar with all controls
- [ ] Build Sidebar with project tree and agent quick access
- [ ] Add theme switching functionality

---

## Phase 4: Core Pages

### 4.1 Start Page (New Project)

**File: `src/pages/StartPage.tsx`**

```typescript
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { Rocket, FolderPlus, Sparkles } from 'lucide-react';
import { useCreateProject } from '@/hooks/useProjects';
import { useAppStore } from '@/stores/appStore';

export function StartPage() {
  const navigate = useNavigate();
  const { setCurrentProject } = useAppStore();
  const createProject = useCreateProject();

  const [projectName, setProjectName] = useState('');
  const [description, setDescription] = useState('');
  const [userRequest, setUserRequest] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    if (!projectName.trim() || !userRequest.trim()) return;

    try {
      // Create project
      const project = await createProject.mutateAsync({
        name: projectName,
        description: description || undefined,
      });

      setCurrentProject(project);

      // Navigate to new session setup
      navigate('/new-session', {
        state: {
          projectId: project.id,
          userRequest
        }
      });
    } catch (error) {
      console.error('Failed to create project:', error);
    }
  };

  return (
    <div className="min-h-full flex items-center justify-center p-8">
      <div className="w-full max-w-2xl">
        {/* Header */}
        <div className="text-center mb-8">
          <div className="inline-flex items-center justify-center w-16 h-16 bg-gradient-to-br from-primary-500 to-primary-700 rounded-2xl mb-4">
            <Rocket className="w-8 h-8 text-white" />
          </div>
          <h1 className="text-3xl font-bold mb-2">Start a New Project</h1>
          <p className="text-gray-600 dark:text-gray-400">
            Describe what you want to build and our AI agents will make it happen.
          </p>
        </div>

        {/* Form */}
        <form onSubmit={handleSubmit} className="card p-6 space-y-6">
          {/* Project Name */}
          <div>
            <label
              htmlFor="projectName"
              className="block text-sm font-medium mb-2"
            >
              Project Name
            </label>
            <div className="relative">
              <FolderPlus className="absolute left-3 top-1/2 -translate-y-1/2 w-5 h-5 text-gray-400" />
              <input
                id="projectName"
                type="text"
                value={projectName}
                onChange={(e) => setProjectName(e.target.value)}
                placeholder="My Awesome Project"
                className="input pl-10"
                required
              />
            </div>
          </div>

          {/* Description (Optional) */}
          <div>
            <label
              htmlFor="description"
              className="block text-sm font-medium mb-2"
            >
              Description <span className="text-gray-400">(optional)</span>
            </label>
            <input
              id="description"
              type="text"
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              placeholder="A brief description of your project"
              className="input"
            />
          </div>

          {/* User Request */}
          <div>
            <label
              htmlFor="userRequest"
              className="block text-sm font-medium mb-2"
            >
              What do you want to build?
            </label>
            <div className="relative">
              <Sparkles className="absolute left-3 top-3 w-5 h-5 text-gray-400" />
              <textarea
                id="userRequest"
                value={userRequest}
                onChange={(e) => setUserRequest(e.target.value)}
                placeholder="Describe your project in detail. For example: Build a REST API with user authentication, CRUD operations for products, and integration with Stripe for payments..."
                className="input pl-10 min-h-[150px] resize-y"
                required
              />
            </div>
            <p className="mt-2 text-sm text-gray-500">
              Be as detailed as possible. The more context you provide, the better results you'll get.
            </p>
          </div>

          {/* Submit */}
          <button
            type="submit"
            disabled={createProject.isPending || !projectName.trim() || !userRequest.trim()}
            className="btn-primary w-full py-3"
          >
            {createProject.isPending ? (
              <>
                <div className="w-5 h-5 border-2 border-white border-t-transparent rounded-full animate-spin mr-2" />
                Creating Project...
              </>
            ) : (
              <>
                <Rocket className="w-5 h-5 mr-2" />
                Start Building
              </>
            )}
          </button>
        </form>

        {/* Info Cards */}
        <div className="mt-8 grid grid-cols-2 gap-4">
          <div className="card p-4">
            <h3 className="font-semibold mb-1">4 Specialized Agents</h3>
            <p className="text-sm text-gray-600 dark:text-gray-400">
              Planner, Builder, QA/QC, and Production Readiness agents work together.
            </p>
          </div>
          <div className="card p-4">
            <h3 className="font-semibold mb-1">Powered by Codex</h3>
            <p className="text-sm text-gray-600 dark:text-gray-400">
              Agents orchestrate codex-cli to write production-ready code.
            </p>
          </div>
        </div>
      </div>
    </div>
  );
}
```

### 4.2 Session Page (Main Chat Interface)

**File: `src/pages/SessionPage.tsx`**

```typescript
import { useEffect, useRef, useState } from 'react';
import { useParams } from 'react-router-dom';
import { Send, Upload, Loader2 } from 'lucide-react';
import { useAppStore } from '@/stores/appStore';
import { useSession, useSessionConversations } from '@/hooks/useSessions';
import { WebSocketClient } from '@/lib/websocket';
import { ChatMessage } from '@/components/chat/ChatMessage';
import { AgentTabs } from '@/components/chat/AgentTabs';
import { WorkflowProgress } from '@/components/chat/WorkflowProgress';
import { FileUpload } from '@/components/chat/FileUpload';
import type { AgentRole, Conversation } from '@/lib/api';

export function SessionPage() {
  const { sessionId } = useParams<{ sessionId: string }>();
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const wsRef = useRef<WebSocketClient | null>(null);

  const {
    currentProject,
    setCurrentSession,
    setWorkflowStatus,
    selectedAgent,
    setSelectedAgent,
    conversations,
    setConversations,
    addMessage,
    workflowStatus,
  } = useAppStore();

  const { data: session } = useSession(sessionId ?? null);
  const { data: serverConversations } = useSessionConversations(sessionId ?? null);

  const [inputValue, setInputValue] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [showUpload, setShowUpload] = useState(false);

  // Set current session when loaded
  useEffect(() => {
    if (session) {
      setCurrentSession(session);
    }
  }, [session, setCurrentSession]);

  // Sync conversations from server
  useEffect(() => {
    if (serverConversations) {
      setConversations(serverConversations);
      // Select the first agent that has messages, or planner by default
      if (!selectedAgent) {
        const agentWithMessages = serverConversations.find(c => c.messages.length > 0);
        setSelectedAgent(agentWithMessages?.agent_type || 'planner');
      }
    }
  }, [serverConversations, setConversations, selectedAgent, setSelectedAgent]);

  // WebSocket connection
  useEffect(() => {
    if (!sessionId) return;

    const ws = new WebSocketClient(sessionId);
    wsRef.current = ws;

    ws.connect().then(() => {
      ws.subscribe((message) => {
        switch (message.type) {
          case 'workflow_progress':
            setWorkflowStatus('running', message.stage);
            break;

          case 'workflow_complete':
            setWorkflowStatus(
              message.result?.final_status === 'completed' ? 'completed' : 'failed'
            );
            break;

          case 'agent_message':
            if (message.agent_role && message.content) {
              // Find the conversation for this agent
              const conv = conversations.find(
                c => c.agent_type === message.agent_role
              );
              if (conv) {
                addMessage(conv.id, {
                  role: 'assistant',
                  content: message.content,
                });
              }
            }
            setIsLoading(false);
            break;

          case 'error':
            console.error('WebSocket error:', message.message);
            setIsLoading(false);
            break;
        }
      });
    });

    return () => {
      ws.disconnect();
    };
  }, [sessionId, setWorkflowStatus, conversations, addMessage]);

  // Auto-scroll to bottom
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [conversations]);

  // Get current conversation
  const currentConversation = conversations.find(
    c => c.agent_type === selectedAgent
  );

  const handleSend = async () => {
    if (!inputValue.trim() || !selectedAgent || !currentProject || !sessionId) return;

    setIsLoading(true);

    // Add user message to UI immediately
    if (currentConversation) {
      addMessage(currentConversation.id, {
        role: 'user',
        content: inputValue,
      });
    }

    // Send via WebSocket
    wsRef.current?.chatWithAgent(
      currentProject.id,
      selectedAgent,
      inputValue
    );

    setInputValue('');
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      handleSend();
    }
  };

  const handleStartWorkflow = () => {
    if (!currentProject || !session || !wsRef.current) return;

    setWorkflowStatus('running', 'planner');
    wsRef.current.startWorkflow(currentProject.id, session.user_request);
  };

  return (
    <div className="h-full flex flex-col">
      {/* Workflow Progress Bar */}
      {(workflowStatus === 'running' || workflowStatus === 'completed') && (
        <WorkflowProgress />
      )}

      {/* Agent Tabs */}
      <AgentTabs
        conversations={conversations}
        selectedAgent={selectedAgent}
        onSelectAgent={setSelectedAgent}
      />

      {/* Messages Area */}
      <div className="flex-1 overflow-auto p-4">
        {session?.status === 'pending' && (
          <div className="flex flex-col items-center justify-center h-full text-center">
            <h2 className="text-xl font-semibold mb-2">Ready to Start</h2>
            <p className="text-gray-600 dark:text-gray-400 mb-4 max-w-md">
              {session.user_request}
            </p>
            <button onClick={handleStartWorkflow} className="btn-primary">
              Start 4-Agent Workflow
            </button>
            <p className="mt-4 text-sm text-gray-500">
              Or chat with individual agents below
            </p>
          </div>
        )}

        {currentConversation?.messages.map((message) => (
          <ChatMessage
            key={message.id}
            message={message}
            agentRole={selectedAgent!}
          />
        ))}

        {isLoading && (
          <div className="flex items-center gap-2 text-gray-500 py-4">
            <Loader2 className="w-4 h-4 animate-spin" />
            <span>Agent is thinking...</span>
          </div>
        )}

        <div ref={messagesEndRef} />
      </div>

      {/* Input Area */}
      <div className="border-t border-gray-200 dark:border-gray-800 p-4">
        {showUpload && currentProject && (
          <FileUpload
            projectId={currentProject.id}
            onClose={() => setShowUpload(false)}
          />
        )}

        <div className="flex items-end gap-2">
          <button
            onClick={() => setShowUpload(!showUpload)}
            className="btn-secondary p-2"
            aria-label="Upload file"
          >
            <Upload className="w-5 h-5" />
          </button>

          <div className="flex-1 relative">
            <textarea
              value={inputValue}
              onChange={(e) => setInputValue(e.target.value)}
              onKeyDown={handleKeyDown}
              placeholder={`Message ${selectedAgent ? AGENT_CONFIG[selectedAgent].label : 'agent'}...`}
              className="input min-h-[44px] max-h-32 resize-none pr-12"
              rows={1}
              disabled={isLoading}
            />
          </div>

          <button
            onClick={handleSend}
            disabled={!inputValue.trim() || isLoading}
            className="btn-primary p-2"
            aria-label="Send message"
          >
            <Send className="w-5 h-5" />
          </button>
        </div>
      </div>
    </div>
  );
}

const AGENT_CONFIG: Record<AgentRole, { label: string }> = {
  planner: { label: 'Planner' },
  builder: { label: 'Builder' },
  qa_qc: { label: 'QA/QC' },
  prod_ready: { label: 'Production' },
};
```

### 4.3 Chat Components

**File: `src/components/chat/ChatMessage.tsx`**

```typescript
import { memo } from 'react';
import { User, Bot } from 'lucide-react';
import { formatDistanceToNow } from 'date-fns';
import type { Message, AgentRole } from '@/lib/api';
import { MarkdownRenderer } from './MarkdownRenderer';

interface ChatMessageProps {
  message: Message;
  agentRole: AgentRole;
}

const AGENT_COLORS: Record<AgentRole, string> = {
  planner: 'bg-purple-500',
  builder: 'bg-amber-500',
  qa_qc: 'bg-emerald-500',
  prod_ready: 'bg-red-500',
};

export const ChatMessage = memo(function ChatMessage({
  message,
  agentRole
}: ChatMessageProps) {
  const isUser = message.role === 'user';

  return (
    <div
      className={`flex gap-3 py-4 animate-fade-in ${isUser ? 'flex-row-reverse' : ''}`}
    >
      {/* Avatar */}
      <div
        className={`
          flex-shrink-0 w-8 h-8 rounded-full flex items-center justify-center
          ${isUser
            ? 'bg-gray-200 dark:bg-gray-700'
            : AGENT_COLORS[agentRole]
          }
        `}
      >
        {isUser ? (
          <User className="w-4 h-4 text-gray-600 dark:text-gray-300" />
        ) : (
          <Bot className="w-4 h-4 text-white" />
        )}
      </div>

      {/* Message Content */}
      <div className={`flex-1 max-w-[80%] ${isUser ? 'text-right' : ''}`}>
        <div
          className={`
            inline-block rounded-2xl px-4 py-2
            ${isUser
              ? 'bg-primary-600 text-white rounded-br-md'
              : 'bg-gray-100 dark:bg-gray-800 rounded-bl-md'
            }
          `}
        >
          {isUser ? (
            <p className="whitespace-pre-wrap">{message.content}</p>
          ) : (
            <MarkdownRenderer content={message.content} />
          )}
        </div>
        <p className="text-xs text-gray-400 mt-1">
          {formatDistanceToNow(new Date(message.created_at), { addSuffix: true })}
        </p>
      </div>
    </div>
  );
});
```

**File: `src/components/chat/MarkdownRenderer.tsx`**

```typescript
import { memo } from 'react';

interface MarkdownRendererProps {
  content: string;
}

export const MarkdownRenderer = memo(function MarkdownRenderer({
  content
}: MarkdownRendererProps) {
  // Simple markdown parsing - for production, use react-markdown
  const renderContent = () => {
    const lines = content.split('\n');
    const elements: JSX.Element[] = [];
    let inCodeBlock = false;
    let codeContent = '';
    let codeLanguage = '';

    lines.forEach((line, index) => {
      // Code blocks
      if (line.startsWith('```')) {
        if (!inCodeBlock) {
          inCodeBlock = true;
          codeLanguage = line.slice(3).trim();
          codeContent = '';
        } else {
          inCodeBlock = false;
          elements.push(
            <pre
              key={index}
              className="bg-terminal-bg text-terminal-text rounded-lg p-4 my-2 overflow-x-auto font-mono text-sm"
            >
              <code>{codeContent}</code>
            </pre>
          );
        }
        return;
      }

      if (inCodeBlock) {
        codeContent += (codeContent ? '\n' : '') + line;
        return;
      }

      // Headers
      if (line.startsWith('### ')) {
        elements.push(
          <h3 key={index} className="text-lg font-semibold mt-4 mb-2">
            {line.slice(4)}
          </h3>
        );
        return;
      }
      if (line.startsWith('## ')) {
        elements.push(
          <h2 key={index} className="text-xl font-semibold mt-4 mb-2">
            {line.slice(3)}
          </h2>
        );
        return;
      }
      if (line.startsWith('# ')) {
        elements.push(
          <h1 key={index} className="text-2xl font-bold mt-4 mb-2">
            {line.slice(2)}
          </h1>
        );
        return;
      }

      // Bullet points
      if (line.startsWith('- ') || line.startsWith('* ')) {
        elements.push(
          <li key={index} className="ml-4">
            {renderInlineStyles(line.slice(2))}
          </li>
        );
        return;
      }

      // Numbered lists
      const numberedMatch = line.match(/^(\d+)\.\s/);
      if (numberedMatch) {
        elements.push(
          <li key={index} className="ml-4 list-decimal">
            {renderInlineStyles(line.slice(numberedMatch[0].length))}
          </li>
        );
        return;
      }

      // Regular paragraph
      if (line.trim()) {
        elements.push(
          <p key={index} className="my-1">
            {renderInlineStyles(line)}
          </p>
        );
      } else {
        elements.push(<br key={index} />);
      }
    });

    return elements;
  };

  const renderInlineStyles = (text: string) => {
    // Bold
    text = text.replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>');
    // Italic
    text = text.replace(/\*(.*?)\*/g, '<em>$1</em>');
    // Inline code
    text = text.replace(/`(.*?)`/g, '<code class="bg-gray-200 dark:bg-gray-700 px-1 rounded">$1</code>');

    return <span dangerouslySetInnerHTML={{ __html: text }} />;
  };

  return <div className="prose dark:prose-invert max-w-none">{renderContent()}</div>;
});
```

**File: `src/components/chat/AgentTabs.tsx`**

```typescript
import { Target, Hammer, CheckCircle, Rocket } from 'lucide-react';
import type { AgentRole, Conversation } from '@/lib/api';

interface AgentTabsProps {
  conversations: Conversation[];
  selectedAgent: AgentRole | null;
  onSelectAgent: (agent: AgentRole) => void;
}

const AGENTS: { role: AgentRole; icon: typeof Target; label: string; color: string }[] = [
  { role: 'planner', icon: Target, label: 'Planner', color: 'agent-planner' },
  { role: 'builder', icon: Hammer, label: 'Builder', color: 'agent-builder' },
  { role: 'qa_qc', icon: CheckCircle, label: 'QA/QC', color: 'agent-qa' },
  { role: 'prod_ready', icon: Rocket, label: 'Production', color: 'agent-prod' },
];

export function AgentTabs({
  conversations,
  selectedAgent,
  onSelectAgent
}: AgentTabsProps) {
  const getMessageCount = (role: AgentRole) => {
    const conv = conversations.find(c => c.agent_type === role);
    return conv?.messages.length || 0;
  };

  const getStatus = (role: AgentRole) => {
    const conv = conversations.find(c => c.agent_type === role);
    return conv?.status;
  };

  return (
    <div className="border-b border-gray-200 dark:border-gray-800">
      <div className="flex">
        {AGENTS.map(({ role, icon: Icon, label, color }) => {
          const isSelected = selectedAgent === role;
          const messageCount = getMessageCount(role);
          const status = getStatus(role);

          return (
            <button
              key={role}
              onClick={() => onSelectAgent(role)}
              className={`
                flex items-center gap-2 px-4 py-3 border-b-2 transition-colors
                ${isSelected
                  ? `border-${color} text-${color}`
                  : 'border-transparent hover:bg-gray-50 dark:hover:bg-gray-800/50'
                }
              `}
            >
              <Icon className={`w-4 h-4 ${isSelected ? `text-${color}` : ''}`} />
              <span className="font-medium">{label}</span>

              {messageCount > 0 && (
                <span className="ml-1 px-1.5 py-0.5 text-xs bg-gray-200 dark:bg-gray-700 rounded-full">
                  {messageCount}
                </span>
              )}

              {status === 'running' && (
                <span className="w-2 h-2 bg-green-500 rounded-full animate-pulse" />
              )}
            </button>
          );
        })}
      </div>
    </div>
  );
}
```

**File: `src/components/chat/WorkflowProgress.tsx`**

```typescript
import { Check, Loader2 } from 'lucide-react';
import { useAppStore } from '@/stores/appStore';
import type { AgentRole } from '@/lib/api';

const WORKFLOW_STAGES: { role: AgentRole; label: string }[] = [
  { role: 'planner', label: 'Planning' },
  { role: 'builder', label: 'Building' },
  { role: 'qa_qc', label: 'Testing' },
  { role: 'prod_ready', label: 'Finalizing' },
];

export function WorkflowProgress() {
  const { workflowStage, workflowStatus } = useAppStore();

  const currentIndex = WORKFLOW_STAGES.findIndex(s => s.role === workflowStage);

  return (
    <div className="bg-white dark:bg-gray-900 border-b border-gray-200 dark:border-gray-800 px-4 py-3">
      <div className="flex items-center justify-between max-w-2xl mx-auto">
        {WORKFLOW_STAGES.map((stage, index) => {
          const isComplete = index < currentIndex || workflowStatus === 'completed';
          const isCurrent = index === currentIndex && workflowStatus === 'running';
          const isPending = index > currentIndex && workflowStatus !== 'completed';

          return (
            <div key={stage.role} className="flex items-center">
              {/* Stage Indicator */}
              <div className="flex items-center">
                <div
                  className={`
                    w-8 h-8 rounded-full flex items-center justify-center
                    ${isComplete
                      ? 'bg-green-500 text-white'
                      : isCurrent
                        ? 'bg-primary-500 text-white'
                        : 'bg-gray-200 dark:bg-gray-700 text-gray-400'
                    }
                  `}
                >
                  {isComplete ? (
                    <Check className="w-4 h-4" />
                  ) : isCurrent ? (
                    <Loader2 className="w-4 h-4 animate-spin" />
                  ) : (
                    <span className="text-sm">{index + 1}</span>
                  )}
                </div>
                <span
                  className={`
                    ml-2 text-sm font-medium
                    ${isCurrent ? 'text-primary-600 dark:text-primary-400' : ''}
                    ${isPending ? 'text-gray-400' : ''}
                  `}
                >
                  {stage.label}
                </span>
              </div>

              {/* Connector Line */}
              {index < WORKFLOW_STAGES.length - 1 && (
                <div
                  className={`
                    w-12 h-0.5 mx-2
                    ${index < currentIndex || workflowStatus === 'completed'
                      ? 'bg-green-500'
                      : 'bg-gray-200 dark:bg-gray-700'
                    }
                  `}
                />
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

**File: `src/components/chat/FileUpload.tsx`**

```typescript
import { useState, useCallback } from 'react';
import { Upload, X, File, Loader2 } from 'lucide-react';
import { filesApi, type ProjectFile } from '@/lib/api';

interface FileUploadProps {
  projectId: string;
  onClose: () => void;
}

export function FileUpload({ projectId, onClose }: FileUploadProps) {
  const [isDragging, setIsDragging] = useState(false);
  const [isUploading, setIsUploading] = useState(false);
  const [uploadedFiles, setUploadedFiles] = useState<ProjectFile[]>([]);

  const handleDragOver = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(true);
  }, []);

  const handleDragLeave = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(false);
  }, []);

  const handleDrop = useCallback(async (e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(false);

    const files = Array.from(e.dataTransfer.files);
    await uploadFiles(files);
  }, [projectId]);

  const handleFileSelect = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const files = Array.from(e.target.files || []);
    await uploadFiles(files);
  };

  const uploadFiles = async (files: File[]) => {
    setIsUploading(true);

    try {
      for (const file of files) {
        const response = await filesApi.upload(projectId, file);
        setUploadedFiles(prev => [...prev, response.data]);
      }
    } catch (error) {
      console.error('Upload failed:', error);
    } finally {
      setIsUploading(false);
    }
  };

  const formatFileSize = (bytes: number) => {
    if (bytes < 1024) return `${bytes} B`;
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
    return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
  };

  return (
    <div className="mb-4 p-4 bg-gray-50 dark:bg-gray-800/50 rounded-lg">
      <div className="flex items-center justify-between mb-3">
        <h3 className="font-medium">Upload Files for Context</h3>
        <button
          onClick={onClose}
          className="p-1 hover:bg-gray-200 dark:hover:bg-gray-700 rounded"
        >
          <X className="w-4 h-4" />
        </button>
      </div>

      {/* Drop Zone */}
      <div
        onDragOver={handleDragOver}
        onDragLeave={handleDragLeave}
        onDrop={handleDrop}
        className={`
          border-2 border-dashed rounded-lg p-6 text-center transition-colors
          ${isDragging
            ? 'border-primary-500 bg-primary-50 dark:bg-primary-900/20'
            : 'border-gray-300 dark:border-gray-600'
          }
        `}
      >
        {isUploading ? (
          <div className="flex items-center justify-center gap-2">
            <Loader2 className="w-5 h-5 animate-spin" />
            <span>Uploading...</span>
          </div>
        ) : (
          <>
            <Upload className="w-8 h-8 mx-auto mb-2 text-gray-400" />
            <p className="text-sm text-gray-600 dark:text-gray-400">
              Drag & drop files here, or{' '}
              <label className="text-primary-600 hover:underline cursor-pointer">
                browse
                <input
                  type="file"
                  multiple
                  onChange={handleFileSelect}
                  className="hidden"
                />
              </label>
            </p>
            <p className="text-xs text-gray-400 mt-1">
              Upload code files, docs, or any context for the agents
            </p>
          </>
        )}
      </div>

      {/* Uploaded Files List */}
      {uploadedFiles.length > 0 && (
        <div className="mt-3 space-y-2">
          {uploadedFiles.map(file => (
            <div
              key={file.id}
              className="flex items-center gap-2 p-2 bg-white dark:bg-gray-800 rounded"
            >
              <File className="w-4 h-4 text-gray-400" />
              <span className="flex-1 text-sm truncate">{file.filename}</span>
              <span className="text-xs text-gray-400">
                {formatFileSize(file.file_size)}
              </span>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

**Checklist:**
- [ ] Create StartPage with project creation form
- [ ] Build SessionPage with full chat interface
- [ ] Implement ChatMessage component with markdown
- [ ] Create AgentTabs for agent switching
- [ ] Build WorkflowProgress indicator
- [ ] Implement FileUpload with drag & drop

---

## Phase 5: Terminal Component

### 5.1 Xterm.js Terminal

**File: `src/components/terminal/TerminalPanel.tsx`**

```typescript
import { useEffect, useRef } from 'react';
import { Terminal } from '@xterm/xterm';
import { FitAddon } from '@xterm/addon-fit';
import { WebLinksAddon } from '@xterm/addon-web-links';
import '@xterm/xterm/css/xterm.css';
import { X, Maximize2, Minimize2 } from 'lucide-react';
import { useAppStore } from '@/stores/appStore';

export function TerminalPanel() {
  const terminalRef = useRef<HTMLDivElement>(null);
  const xtermRef = useRef<Terminal | null>(null);
  const fitAddonRef = useRef<FitAddon | null>(null);

  const {
    terminalOutput,
    setTerminalVisible,
    clearTerminalOutput
  } = useAppStore();

  // Initialize terminal
  useEffect(() => {
    if (!terminalRef.current) return;

    const terminal = new Terminal({
      theme: {
        background: '#1a1b26',
        foreground: '#a9b1d6',
        cursor: '#c0caf5',
        cursorAccent: '#1a1b26',
        selectionBackground: '#33467c',
        black: '#32344a',
        red: '#f7768e',
        green: '#9ece6a',
        yellow: '#e0af68',
        blue: '#7aa2f7',
        magenta: '#ad8ee6',
        cyan: '#449dab',
        white: '#787c99',
        brightBlack: '#444b6a',
        brightRed: '#ff7a93',
        brightGreen: '#b9f27c',
        brightYellow: '#ff9e64',
        brightBlue: '#7da6ff',
        brightMagenta: '#bb9af7',
        brightCyan: '#0db9d7',
        brightWhite: '#acb0d0',
      },
      fontFamily: '"JetBrains Mono", "Fira Code", monospace',
      fontSize: 13,
      lineHeight: 1.4,
      cursorBlink: true,
      cursorStyle: 'bar',
      scrollback: 5000,
      convertEol: true,
    });

    const fitAddon = new FitAddon();
    const webLinksAddon = new WebLinksAddon();

    terminal.loadAddon(fitAddon);
    terminal.loadAddon(webLinksAddon);

    terminal.open(terminalRef.current);
    fitAddon.fit();

    xtermRef.current = terminal;
    fitAddonRef.current = fitAddon;

    // Welcome message
    terminal.writeln('\x1b[1;34m╔══════════════════════════════════════════╗\x1b[0m');
    terminal.writeln('\x1b[1;34m║\x1b[0m    \x1b[1;36mCodex Agent Terminal\x1b[0m                  \x1b[1;34m║\x1b[0m');
    terminal.writeln('\x1b[1;34m║\x1b[0m    Live output from codex-cli            \x1b[1;34m║\x1b[0m');
    terminal.writeln('\x1b[1;34m╚══════════════════════════════════════════╝\x1b[0m');
    terminal.writeln('');

    // Handle resize
    const resizeObserver = new ResizeObserver(() => {
      fitAddon.fit();
    });
    resizeObserver.observe(terminalRef.current);

    return () => {
      resizeObserver.disconnect();
      terminal.dispose();
    };
  }, []);

  // Write new output
  useEffect(() => {
    if (!xtermRef.current) return;

    const lastLines = terminalOutput.slice(-100);
    lastLines.forEach(line => {
      // Color-code different types of output
      let coloredLine = line;

      if (line.includes('[stderr]')) {
        coloredLine = `\x1b[31m${line}\x1b[0m`; // Red for errors
      } else if (line.includes('✓') || line.includes('success') || line.includes('completed')) {
        coloredLine = `\x1b[32m${line}\x1b[0m`; // Green for success
      } else if (line.includes('⚠') || line.includes('warning')) {
        coloredLine = `\x1b[33m${line}\x1b[0m`; // Yellow for warnings
      } else if (line.startsWith('>') || line.startsWith('$')) {
        coloredLine = `\x1b[36m${line}\x1b[0m`; // Cyan for commands
      }

      xtermRef.current?.writeln(coloredLine);
    });
  }, [terminalOutput]);

  return (
    <div className="h-full flex flex-col bg-terminal-bg">
      {/* Header */}
      <div className="flex items-center justify-between px-4 py-2 bg-gray-800 border-b border-gray-700">
        <div className="flex items-center gap-2">
          <div className="flex gap-1.5">
            <div className="w-3 h-3 rounded-full bg-red-500" />
            <div className="w-3 h-3 rounded-full bg-yellow-500" />
            <div className="w-3 h-3 rounded-full bg-green-500" />
          </div>
          <span className="text-sm text-gray-400 ml-2">Codex Terminal</span>
        </div>

        <div className="flex items-center gap-1">
          <button
            onClick={clearTerminalOutput}
            className="p-1 text-gray-400 hover:text-white hover:bg-gray-700 rounded"
            title="Clear terminal"
          >
            <span className="text-xs">Clear</span>
          </button>
          <button
            onClick={() => setTerminalVisible(false)}
            className="p-1 text-gray-400 hover:text-white hover:bg-gray-700 rounded"
            title="Close terminal"
          >
            <X className="w-4 h-4" />
          </button>
        </div>
      </div>

      {/* Terminal */}
      <div ref={terminalRef} className="flex-1 p-2" />
    </div>
  );
}
```

**Checklist:**
- [ ] Install and configure Xterm.js
- [ ] Implement TerminalPanel component
- [ ] Add terminal theming
- [ ] Connect to WebSocket for live output
- [ ] Add terminal controls (clear, close, maximize)

---

## Phase 6: Router Setup

### 6.1 Application Routes

**File: `src/App.tsx`**

```typescript
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@/lib/queryClient';
import { RootLayout } from '@/components/layout/RootLayout';
import { StartPage } from '@/pages/StartPage';
import { SessionPage } from '@/pages/SessionPage';
import { AgentChatPage } from '@/pages/AgentChatPage';
import { ProjectsPage } from '@/pages/ProjectsPage';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <Routes>
          <Route element={<RootLayout />}>
            <Route path="/" element={<Navigate to="/new-project" replace />} />
            <Route path="/new-project" element={<StartPage />} />
            <Route path="/projects" element={<ProjectsPage />} />
            <Route path="/session/:sessionId" element={<SessionPage />} />
            <Route path="/chat/:agentRole" element={<AgentChatPage />} />
          </Route>
        </Routes>
      </BrowserRouter>
    </QueryClientProvider>
  );
}

export default App;
```

### 6.2 Additional Pages

**File: `src/pages/ProjectsPage.tsx`**

```typescript
import { useNavigate } from 'react-router-dom';
import { FolderOpen, Plus, Trash2, Calendar } from 'lucide-react';
import { useProjects, useDeleteProject } from '@/hooks/useProjects';
import { useAppStore } from '@/stores/appStore';
import { formatDistanceToNow } from 'date-fns';

export function ProjectsPage() {
  const navigate = useNavigate();
  const { setCurrentProject } = useAppStore();
  const { data: projects, isLoading } = useProjects();
  const deleteProject = useDeleteProject();

  const handleSelectProject = (project: any) => {
    setCurrentProject(project);
    navigate(`/projects/${project.id}`);
  };

  const handleDeleteProject = async (e: React.MouseEvent, projectId: string) => {
    e.stopPropagation();
    if (confirm('Are you sure you want to delete this project?')) {
      await deleteProject.mutateAsync(projectId);
    }
  };

  return (
    <div className="p-8 max-w-5xl mx-auto">
      <div className="flex items-center justify-between mb-8">
        <div>
          <h1 className="text-2xl font-bold">Projects</h1>
          <p className="text-gray-600 dark:text-gray-400">
            Manage your development projects
          </p>
        </div>
        <button
          onClick={() => navigate('/new-project')}
          className="btn-primary"
        >
          <Plus className="w-4 h-4 mr-2" />
          New Project
        </button>
      </div>

      {isLoading ? (
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
          {[1, 2, 3].map(i => (
            <div key={i} className="card h-40 animate-pulse bg-gray-100 dark:bg-gray-800" />
          ))}
        </div>
      ) : projects?.length === 0 ? (
        <div className="card p-12 text-center">
          <FolderOpen className="w-12 h-12 mx-auto mb-4 text-gray-400" />
          <h3 className="text-lg font-semibold mb-2">No projects yet</h3>
          <p className="text-gray-600 dark:text-gray-400 mb-4">
            Create your first project to get started
          </p>
          <button
            onClick={() => navigate('/new-project')}
            className="btn-primary"
          >
            Create Project
          </button>
        </div>
      ) : (
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
          {projects?.map(project => (
            <div
              key={project.id}
              onClick={() => handleSelectProject(project)}
              className="card p-4 cursor-pointer hover:shadow-md transition-shadow group"
            >
              <div className="flex items-start justify-between mb-3">
                <div className="w-10 h-10 bg-primary-100 dark:bg-primary-900/30 rounded-lg flex items-center justify-center">
                  <FolderOpen className="w-5 h-5 text-primary-600 dark:text-primary-400" />
                </div>
                <button
                  onClick={(e) => handleDeleteProject(e, project.id)}
                  className="p-1 opacity-0 group-hover:opacity-100 hover:bg-red-100 dark:hover:bg-red-900/30 rounded transition-all"
                >
                  <Trash2 className="w-4 h-4 text-red-500" />
                </button>
              </div>

              <h3 className="font-semibold mb-1 truncate">{project.name}</h3>
              {project.description && (
                <p className="text-sm text-gray-600 dark:text-gray-400 line-clamp-2 mb-3">
                  {project.description}
                </p>
              )}

              <div className="flex items-center text-xs text-gray-400 mt-auto pt-2 border-t border-gray-100 dark:border-gray-800">
                <Calendar className="w-3 h-3 mr-1" />
                {formatDistanceToNow(new Date(project.updated_at), { addSuffix: true })}
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

**File: `src/pages/AgentChatPage.tsx`**

```typescript
import { useEffect, useRef, useState } from 'react';
import { useParams, useNavigate } from 'react-router-dom';
import { Send, ArrowLeft } from 'lucide-react';
import { useAppStore } from '@/stores/appStore';
import { WebSocketClient } from '@/lib/websocket';
import { ChatMessage } from '@/components/chat/ChatMessage';
import type { AgentRole, Message } from '@/lib/api';

const AGENT_INFO: Record<AgentRole, { name: string; description: string }> = {
  planner: {
    name: 'Planner Agent',
    description: 'Expert business analyst that creates detailed requirements and AGENTS.md specifications.',
  },
  builder: {
    name: 'Builder Agent',
    description: 'Expert developer that implements requirements step by step using codex-cli.',
  },
  qa_qc: {
    name: 'QA/QC Agent',
    description: 'Quality assurance expert that tests, debugs, and validates the implementation.',
  },
  prod_ready: {
    name: 'Production Readiness Agent',
    description: 'DevOps expert that ensures the build is ready for production deployment.',
  },
};

export function AgentChatPage() {
  const { agentRole } = useParams<{ agentRole: AgentRole }>();
  const navigate = useNavigate();
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const wsRef = useRef<WebSocketClient | null>(null);

  const { currentProject, currentSession } = useAppStore();

  const [messages, setMessages] = useState<Message[]>([]);
  const [inputValue, setInputValue] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  const agentInfo = agentRole ? AGENT_INFO[agentRole] : null;

  useEffect(() => {
    if (!currentProject || !currentSession) {
      navigate('/new-project');
      return;
    }

    const ws = new WebSocketClient(currentSession.id);
    wsRef.current = ws;

    ws.connect().then(() => {
      ws.subscribe((message) => {
        if (message.type === 'agent_message' && message.content) {
          setMessages(prev => [...prev, {
            id: crypto.randomUUID(),
            role: 'assistant',
            content: message.content,
            created_at: new Date().toISOString(),
          }]);
          setIsLoading(false);
        }
      });
    });

    return () => ws.disconnect();
  }, [currentProject, currentSession, navigate]);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const handleSend = () => {
    if (!inputValue.trim() || !agentRole || !currentProject || !wsRef.current) return;

    setIsLoading(true);

    // Add user message
    setMessages(prev => [...prev, {
      id: crypto.randomUUID(),
      role: 'user',
      content: inputValue,
      created_at: new Date().toISOString(),
    }]);

    // Send to agent
    wsRef.current.chatWithAgent(currentProject.id, agentRole, inputValue);
    setInputValue('');
  };

  if (!agentRole || !agentInfo) {
    return <div>Invalid agent</div>;
  }

  return (
    <div className="h-full flex flex-col">
      {/* Header */}
      <div className="border-b border-gray-200 dark:border-gray-800 px-4 py-3">
        <div className="flex items-center gap-3">
          <button
            onClick={() => navigate(-1)}
            className="p-1 hover:bg-gray-100 dark:hover:bg-gray-800 rounded"
          >
            <ArrowLeft className="w-5 h-5" />
          </button>
          <div>
            <h2 className="font-semibold">{agentInfo.name}</h2>
            <p className="text-sm text-gray-600 dark:text-gray-400">
              {agentInfo.description}
            </p>
          </div>
        </div>
      </div>

      {/* Messages */}
      <div className="flex-1 overflow-auto p-4">
        {messages.length === 0 ? (
          <div className="h-full flex items-center justify-center text-center">
            <div>
              <p className="text-gray-600 dark:text-gray-400 mb-2">
                Start a conversation with {agentInfo.name}
              </p>
              <p className="text-sm text-gray-400">
                This agent will use codex-cli to help with your request
              </p>
            </div>
          </div>
        ) : (
          messages.map(message => (
            <ChatMessage
              key={message.id}
              message={message}
              agentRole={agentRole}
            />
          ))
        )}
        <div ref={messagesEndRef} />
      </div>

      {/* Input */}
      <div className="border-t border-gray-200 dark:border-gray-800 p-4">
        <div className="flex gap-2">
          <textarea
            value={inputValue}
            onChange={(e) => setInputValue(e.target.value)}
            onKeyDown={(e) => {
              if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault();
                handleSend();
              }
            }}
            placeholder={`Ask ${agentInfo.name}...`}
            className="input flex-1 min-h-[44px] max-h-32 resize-none"
            rows={1}
            disabled={isLoading}
          />
          <button
            onClick={handleSend}
            disabled={!inputValue.trim() || isLoading}
            className="btn-primary px-4"
          >
            <Send className="w-5 h-5" />
          </button>
        </div>
      </div>
    </div>
  );
}
```

**Checklist:**
- [ ] Set up React Router with all routes
- [ ] Create ProjectsPage for project list
- [ ] Implement AgentChatPage for direct agent chat
- [ ] Add navigation between pages

---

## Phase 7: Build & Deploy

### 7.1 Vite Configuration

**File: `vite.config.ts`**

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
      '/ws': {
        target: 'ws://localhost:8000',
        ws: true,
      },
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: false,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          ui: ['@xterm/xterm', 'lucide-react'],
        },
      },
    },
  },
});
```

### 7.2 Environment Configuration

**File: `.env.example`**

```bash
VITE_API_URL=http://localhost:8000/api
VITE_WS_URL=ws://localhost:8000/ws
```

### 7.3 Build Commands

```bash
# Development
npm run dev

# Production build
npm run build

# Preview production build
npm run preview

# Type checking
npm run typecheck

# Linting
npm run lint
```

**Checklist:**
- [ ] Configure Vite with path aliases
- [ ] Set up proxy for development
- [ ] Configure production build optimization
- [ ] Create environment configuration

---

## Project Structure

```
frontend/
├── public/
│   └── favicon.ico
├── src/
│   ├── components/
│   │   ├── chat/
│   │   │   ├── AgentTabs.tsx
│   │   │   ├── ChatMessage.tsx
│   │   │   ├── FileUpload.tsx
│   │   │   ├── MarkdownRenderer.tsx
│   │   │   └── WorkflowProgress.tsx
│   │   ├── layout/
│   │   │   ├── Navbar.tsx
│   │   │   ├── RootLayout.tsx
│   │   │   └── Sidebar.tsx
│   │   └── terminal/
│   │       └── TerminalPanel.tsx
│   ├── hooks/
│   │   ├── useProjects.ts
│   │   └── useSessions.ts
│   ├── lib/
│   │   ├── api.ts
│   │   ├── queryClient.ts
│   │   └── websocket.ts
│   ├── pages/
│   │   ├── AgentChatPage.tsx
│   │   ├── ProjectsPage.tsx
│   │   ├── SessionPage.tsx
│   │   └── StartPage.tsx
│   ├── stores/
│   │   └── appStore.ts
│   ├── App.tsx
│   ├── index.css
│   └── main.tsx
├── .env.example
├── index.html
├── package.json
├── tailwind.config.js
├── tsconfig.json
└── vite.config.ts
```

---

## Implementation Checklist

### Phase 1: Setup
- [ ] Initialize Vite project
- [ ] Install all dependencies
- [ ] Configure Tailwind CSS
- [ ] Set up path aliases

### Phase 2: Infrastructure
- [ ] Create API client
- [ ] Implement WebSocket client
- [ ] Set up Zustand store
- [ ] Configure React Query

### Phase 3: Layout
- [ ] Build RootLayout
- [ ] Create Navbar
- [ ] Implement Sidebar
- [ ] Add theme switching

### Phase 4: Pages
- [ ] Create StartPage
- [ ] Build SessionPage
- [ ] Implement chat components
- [ ] Add file upload

### Phase 5: Terminal
- [ ] Configure Xterm.js
- [ ] Build TerminalPanel
- [ ] Connect to live output

### Phase 6: Routing
- [ ] Set up React Router
- [ ] Create all pages
- [ ] Add navigation

### Phase 7: Deploy
- [ ] Configure build
- [ ] Optimize bundle
- [ ] Deploy to production

---

## Key Features Summary

1. **Modern UI**: Clean, responsive design with dark mode support
2. **Project Management**: Create, view, and manage multiple projects
3. **Chat Interface**: Real-time chat with individual agents
4. **Workflow Visualization**: Progress indicator for 4-agent workflow
5. **Terminal Streaming**: Live codex-cli output with Xterm.js
6. **File Upload**: Drag & drop file upload for context
7. **Conversation History**: Grouped by projects with search
8. **WebSocket Communication**: Real-time updates and streaming
