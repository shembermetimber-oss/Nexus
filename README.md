
import React, { useState, useEffect, useRef } from 'react';
import { Terminal } from './components/Terminal';
import { PhishingLab } from './components/PhishingLab';
import { AIBuilderLab } from './components/AIBuilderLab';
import { LLMArchitectLab } from './components/LLMArchitectLab';
import { RobloxLab } from './components/RobloxLab';
import { SingularityLab } from './components/SingularityLab';
import { PhonkFM } from './components/PhonkFM';
import { LiveVoice } from './components/LiveVoice';
import { DeploymentGuide } from './components/DeploymentGuide';
import { LESSONS } from './constants';
import { Lesson, ChatMessage, LessonCategory, GroundingSource } from './types';
import { getMentorStream } from './services/geminiService';
import { GeminiLiveService } from './services/geminiLiveService';
import { 
  ShieldCheck, 
  Cpu, 
  MessageSquare, 
  ChevronRight, 
  Lock, 
  Activity, 
  BookOpen, 
  Zap,
  Play,
  BrainCircuit,
  LayoutDashboard,
  Layers,
  Sparkles,
  Command,
  Globe,
  ExternalLink,
  Search,
  Radar,
  Box,
  Flame,
  Volume2,
  Mic,
  Fingerprint,
  LogIn,
  MapPin,
  Navigation,
  AlertTriangle,
  FileSearch,
  History,
  Info,
  RefreshCw,
  Terminal as TerminalIcon,
  Send,
  Menu,
  X as CloseIcon,
  Trash2,
  Clock,
  GitBranch,
  Github,
  CloudUpload,
  CheckCircle2,
  Book,
  Copy,
  Check,
  ClipboardCheck,
  User,
  Download
} from 'lucide-react';

declare global {
  interface AIStudio {
    hasSelectedApiKey: () => Promise<boolean>;
    openSelectKey: () => Promise<void>;
  }
  interface Window {
    aistudio?: AIStudio;
  }
}

const MessageContent: React.FC<{ text: string }> = ({ text }) => {
  const [copiedIndex, setCopiedIndex] = useState<number | null>(null);

  const handleCopy = (code: string, index: number) => {
    navigator.clipboard.writeText(code);
    setCopiedIndex(index);
    setTimeout(() => setCopiedIndex(null), 2000);
  };

  const parts = text.split(/(```[\s\S]*?```)/g);

  return (
    <div className="space-y-4">
      {parts.map((part, i) => {
        if (part.startsWith('```')) {
          const match = part.match(/```(\w+)?\n?([\s\S]*?)```/);
          const lang = match?.[1] || 'protocol';
          const code = (match?.[2] || '').trim();

          return (
            <div key={i} className="my-8 rounded-2xl overflow-hidden border border-white/10 bg-black/60 shadow-2xl animate-in fade-in zoom-in-95 duration-300">
              <div className="flex items-center justify-between px-6 py-4 bg-zinc-900/90 border-b border-white/5 backdrop-blur-md">
                <div className="flex items-center gap-3">
                  <TerminalIcon className="w-4 h-4 text-indigo-400" />
                  <span className="text-[10px] font-black uppercase tracking-[0.3em] text-zinc-400">{lang}_source</span>
                </div>
                <button 
                  onClick={() => handleCopy(code, i)}
                  className={`flex items-center gap-2 px-4 py-2 rounded-xl transition-all border font-black uppercase tracking-widest text-[9px] active:scale-95 ${
                    copiedIndex === i 
                    ? 'bg-emerald-500/20 border-emerald-500/40 text-emerald-400' 
                    : 'bg-indigo-500/10 border-indigo-500/20 text-indigo-400 hover:bg-indigo-500/20'
                  }`}
                >
                  {copiedIndex === i ? (
                    <>
                      <ClipboardCheck className="w-3.5 h-3.5" />
                      Protocol_Copied
                    </>
                  ) : (
                    <>
                      <Copy className="w-3.5 h-3.5" />
                      Extract_All_Code
                    </>
                  )}
                </button>
              </div>
              <pre className="p-8 mono text-[13px] text-zinc-300 overflow-x-auto custom-scrollbar selection:bg-indigo-500/40 leading-relaxed">
                <code>{code}</code>
              </pre>
            </div>
          );
        }

        const boldParts = part.split(/(\*\*.*?\*\*)/g);
        return (
          <div key={i} className="leading-relaxed">
            {boldParts.map((bp, j) => {
              if (bp.startsWith('**') && bp.endsWith('**')) {
                const content = bp.slice(2, -2);
                return (
                  <strong key={j} className="text-indigo-400 font-black text-glow-indigo mx-0.5 whitespace-nowrap">
                    {content}
                  </strong>
                );
              }
              return <span key={j} dangerouslySetInnerHTML={{ __html: bp.replace(/\n/g, '<br/>') }} />;
            })}
          </div>
        );
      })}
    </div>
  );
};

const App: React.FC = () => {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isLoggingIn, setIsLoggingIn] = useState(false);
  const [activeLesson, setActiveLesson] = useState<Lesson | null>(null);
  const [labMode, setLabMode] = useState(false);
  const [isSyncing, setIsSyncing] = useState(false);
  const [lastCommit, setLastCommit] = useState('HEAD -> 4f8a2b');
  const [showGuide, setShowGuide] = useState(false);
  
  const [chatHistory, setChatHistory] = useState<ChatMessage[]>(() => {
    const saved = localStorage.getItem('nexus_chat_history');
    return saved ? JSON.parse(saved) : [
      { role: 'model', text: 'NEXUS Unified Core initialized. All protocols active, **Commander**. Standing by for directives.' }
    ];
  });

  const [userInput, setUserInput] = useState('');
  const [isTyping, setIsTyping] = useState(false);
  const [isLiveActive, setIsLiveActive] = useState(false);
  const [isSpeaking, setIsSpeaking] = useState(false);
  const [liveTranscript, setLiveTranscript] = useState('');
  const [permissionError, setPermissionError] = useState<string | null>(null);
  const [commandLog, setCommandLog] = useState<{name: string, args: any} | null>(null);
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  
  const scrollContainerRef = useRef<HTMLDivElement>(null);
  const chatEndRef = useRef<HTMLDivElement>(null);
  const liveService = useRef<GeminiLiveService | null>(null);
  const messageRefs = useRef<{[key: number]: HTMLDivElement | null}>({});

  useEffect(() => {
    localStorage.setItem('nexus_chat_history', JSON.stringify(chatHistory));
  }, [chatHistory]);

  useEffect(() => {
    const checkAuth = async () => {
      if (typeof window.aistudio !== 'undefined' && window.aistudio) {
        const hasKey = await window.aistudio.hasSelectedApiKey();
        if (hasKey) setIsAuthenticated(true);
      }
    };
    checkAuth();
  }, []);

  useEffect(() => {
    if (chatEndRef.current && !labMode) {
      chatEndRef.current.scrollIntoView({ behavior: 'smooth' });
    }
  }, [chatHistory.length, isTyping, labMode]);

  const scrollToMessage = (index: number) => {
    messageRefs.current[index]?.scrollIntoView({ behavior: 'smooth', block: 'center' });
    setIsSidebarOpen(false);
  };

  const handleSync = () => {
    setIsSyncing(true);
    setTimeout(() => {
      setIsSyncing(false);
      const newHash = Math.random().toString(16).substring(2, 8);
      setLastCommit(`HEAD -> ${newHash}`);
    }, 2500);
  };

  const clearHistory = () => {
    const initialMsg: ChatMessage = { role: 'model', text: 'Neural Archive purged. Memory modules cleared. System ready for fresh directives.' };
    setChatHistory([initialMsg]);
    setActiveLesson(null);
    setLabMode(false);
  };

  const handleToolExecution = async (fc: any) => {
    setCommandLog({ name: fc.name, args: fc.args });
    setTimeout(() => setCommandLog(null), 3000);

    switch (fc.name) {
      case 'launch_tactical_lab':
        const lesson = LESSONS.find(l => l.id === fc.args.lesson_id);
        if (lesson) {
          setActiveLesson(lesson);
          setLabMode(true);
          return `Lab ${lesson.title} deployed.`;
        }
        return "Target lesson ID on command.";
      case 'execute_network_scan':
        return `Initiating tactical scan for: ${fc.args.query}`;
      default:
        return "Unknown command.";
    }
  };

  const handleGoogleLogin = async () => {
    setIsLoggingIn(true);
    try {
      if (typeof window.aistudio !== 'undefined' && window.aistudio) {
        await window.aistudio.openSelectKey();
        setIsAuthenticated(true);
      } else {
        setTimeout(() => setIsAuthenticated(true), 1000);
      }
    } catch (err) {
      console.error("Authentication failed", err);
    } finally {
      setIsLoggingIn(false);
    }
  };

  const toggleLive = async () => {
    setPermissionError(null);
    if (isLiveActive) {
      liveService.current?.disconnect();
      setIsLiveActive(false);
    } else {
      try {
        if (!liveService.current) liveService.current = new GeminiLiveService();
        setIsLiveActive(true);
        await liveService.current.connect({
          onTranscript: (text, role) => setLiveTranscript(text),
          onAudioStart: () => setIsSpeaking(true),
          onAudioEnd: () => setIsSpeaking(false),
          onToolCall: handleToolExecution,
          onError: (err) => {
            if (err.includes("Requested entity was not found")) {
               setIsAuthenticated(false);
               setPermissionError("Project authentication lost. Please log in again.");
            } else {
               setPermissionError(err);
            }
            setIsLiveActive(false);
          }
        });
      } catch (err: any) {
        setPermissionError(err.message || "Failed Link");
        setIsLiveActive(false);
      }
    }
  };

  const handleSendMessage = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!userInput.trim()) return;

    const input = userInput;
    const userMsg: ChatMessage = { role: 'user', text: input };
    
    setChatHistory(prev => [...prev, userMsg, { role: 'model', text: '' }]);
    setUserInput('');
    setIsTyping(true);

    const apiHistory = chatHistory.map(msg => ({
      role: msg.role,
      parts: [{ text: msg.text }]
    }));

    try {
      const response = await getMentorStream(input, apiHistory, (streamedText) => {
        setChatHistory(prev => {
          const newHistory = [...prev];
          const lastMsg = newHistory[newHistory.length - 1];
          if (lastMsg && lastMsg.role === 'model') {
            lastMsg.text = streamedText;
          }
          return newHistory;
        });
      });
      
      if (response.functionCalls) {
        for (const fc of response.functionCalls) {
          await handleToolExecution(fc);
        }
      }

      setChatHistory(prev => {
        const newHistory = [...prev];
        const lastMsg = newHistory[newHistory.length - 1];
        if (lastMsg && lastMsg.role === 'model') {
          lastMsg.text = response.text || "";
          lastMsg.sources = response.sources;
        }
        return newHistory;
      });

    } catch (err: any) {
      if (err.message?.includes("Requested entity was not found")) {
        setIsAuthenticated(false);
      }
      setChatHistory(prev => [...prev, { role: 'model', text: "Critical System Error: Authentication revoked." }]);
    } finally {
      setIsTyping(false);
    }
  };

  const renderLab = () => {
    if (!activeLesson) return null;
    switch (activeLesson.labType) {
      case 'phishing':
        return <PhishingLab onClose={() => setLabMode(false)} onAttempt={(u, p) => setChatHistory(prev => [...prev, { role: 'model', text: `Data Intercepted. Captured: **${u}**` }])} />;
      case 'ai-builder':
        return <AIBuilderLab onClose={() => setLabMode(false)} onComplete={(acc) => setChatHistory(prev => [...prev, { role: 'model', text: `Synaptic synchronization at **${acc.toFixed(2)}%** accuracy.` }])} />;
      case 'llm-architect':
        return <LLMArchitectLab onClose={() => setLabMode(false)} />;
      case 'roblox-plugin':
        return <RobloxLab onClose={() => setLabMode(false)} />;
      case 'quantum-sim':
        return <SingularityLab onClose={() => setLabMode(false)} type="quantum" />;
      case 'swarm-sandbox':
        return <SingularityLab onClose={() => setLabMode(false)} type="swarm" />;
      default:
        return null;
    }
  };

  if (!isAuthenticated) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-black relative overflow-hidden">
        <div className="absolute inset-0 bg-indigo-500/5 blur-[150px] rounded-full animate-pulse"></div>
        <div className="relative z-10 w-full max-sm px-6">
          <div className="p-10 glass-panel rounded-[2.5rem] border border-indigo-500/20 space-y-12 shadow-2xl">
            <div className="text-center space-y-5">
               <div className="w-24 h-24 bg-white/5 rounded-[2rem] flex items-center justify-center mx-auto mb-8 border border-indigo-500/30 cyber-glow text-indigo-400">
                  <Command className="w-12 h-12" />
               </div>
               <h1 className="text-4xl font-black text-white tracking-[0.2em] uppercase leading-none text-glow-indigo">Nexus</h1>
               <p className="text-zinc-500 text-[10px] font-black uppercase tracking-[0.5em]">Command Authenticator</p>
            </div>
            <button 
              onClick={handleGoogleLogin}
              disabled={isLoggingIn}
              className="w-full py-5 bg-indigo-600 text-white rounded-[1.5rem] font-black uppercase text-[11px] tracking-widest flex items-center justify-center gap-3 hover:bg-indigo-500 transition-all disabled:opacity-50 shadow-[0_0_20px_rgba(99,102,241,0.3)]"
            >
              {isLoggingIn ? <RefreshCw className="w-5 h-5 animate-spin" /> : <LogIn className="w-5 h-5" />}
              Synchronize Link
            </button>
            <div className="text-center space-y-6">
              <div className="pt-4 animate-pulse">
                <p className="text-[10px] font-black text-indigo-400/60 uppercase tracking-[0.5em] mono">shembermetim created me</p>
              </div>
              <a href="https://ai.google.dev/gemini-api/docs/billing" target="_blank" className="text-[9px] mono text-indigo-400/50 font-bold uppercase tracking-[0.3em] hover:text-indigo-400 transition-colors block">
                Billing Docs <ExternalLink className="inline w-2.5 h-2.5 ml-1" />
              </a>
            </div>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen flex flex-col relative overflow-hidden bg-black selection:bg-indigo-500/30">
      <div className="fixed inset-0 pointer-events-none opacity-[0.03] bg-indigo-500 blur-[180px]"></div>

      {commandLog && (
        <div className="fixed top-24 left-1/2 -translate-x-1/2 z-[200] px-8 py-4 glass-panel border border-indigo-500/50 rounded-2xl shadow-2xl animate-in slide-in-from-top-6 duration-500">
           <div className="flex items-center gap-4 text-[11px] font-black uppercase tracking-[0.2em] text-indigo-400">
              <div className="w-2 h-2 rounded-full bg-indigo-400 animate-pulse shadow-[0_0_8px_rgba(99,102,241,0.8)]"></div>
              PROTOCOL_EXEC: {commandLog.name}
           </div>
        </div>
      )}

      {showGuide && <DeploymentGuide onClose={() => setShowGuide(false)} />}

      <nav className="h-16 border-b border-white/5 bg-black/80 backdrop-blur-2xl sticky top-0 z-[110] flex items-center px-4 md:px-8 justify-between shadow-lg">
        <div className="flex items-center gap-4">
          <button onClick={() => setIsSidebarOpen(!isSidebarOpen)} className="p-2.5 hover:bg-white/5 rounded-xl xl:hidden text-zinc-400 transition-colors">
             {isSidebarOpen ? <CloseIcon className="w-5 h-5" /> : <Menu className="w-5 h-5" />}
          </button>
          <div className="flex items-center gap-4 group cursor-pointer" onClick={() => { setActiveLesson(null); setLabMode(false); }}>
            <div className="p-2 rounded-xl bg-indigo-500/10 border border-indigo-500/20 group-hover:border-indigo-500/50 transition-all">
              <Command className="w-5 h-5 text-indigo-400 cyber-glow" />
            </div>
            <div className="flex flex-col">
              <span className="text-[12px] font-black tracking-[0.3em] uppercase hidden sm:block text-glow-indigo leading-tight">
                NEXUS <span className="text-zinc-600">ARCHITECT</span>
              </span>
              <span className="text-[8px] font-black text-indigo-400/50 uppercase tracking-[0.2em] hidden sm:block mono">AUTH_ID: shembermetim</span>
            </div>
          </div>
        </div>

        <div className="text-[10px] font-black uppercase tracking-[0.5em] text-zinc-700 hidden lg:block animate-pulse-neon">
           Secure Neural Pipeline Established
        </div>

        <div className="flex items-center gap-3 md:gap-6">
          <a 
            href="https://github.com" 
            target="_blank" 
            title="Source Repository" 
            className="p-2.5 rounded-full bg-zinc-900 border border-white/5 text-zinc-500 hover:text-rose-500 transition-all hover:scale-110 shadow-[0_0_10px_rgba(244,63,94,0.1)]"
          >
            <Github className="w-5 h-5" />
          </a>
          <button onClick={toggleLive} title="Voice Link" className={`p-2.5 rounded-full border transition-all duration-500 ${isLiveActive ? 'bg-indigo-600 border-indigo-400 text-white shadow-[0_0_20px_rgba(99,102,241,0.6)]' : 'bg-zinc-900 border-white/5 text-zinc-500 hover:text-zinc-300'}`}>
            <Mic className={`w-4.5 h-4.5 ${isLiveActive ? 'animate-pulse' : ''}`} />
          </button>
          <div className="h-8 w-px bg-white/5 mx-1 hidden sm:block"></div>
          <button onClick={() => setIsAuthenticated(false)} title="Logout" className="p-2.5 text-zinc-600 hover:text-rose-500 transition-all hover:scale-110">
            <LogIn className="w-5 h-5 rotate-180" />
          </button>
        </div>
      </nav>

      <main className="flex-1 flex overflow-hidden relative">
        <aside className={`fixed inset-y-0 left-0 z-[105] w-80 bg-black border-r border-white/5 transition-transform duration-500 ease-out transform ${isSidebarOpen ? 'translate-x-0' : '-translate-x-full'} xl:relative xl:translate-x-0 flex flex-col overflow-hidden`}>
           <div className="flex-1 overflow-y-auto p-6 space-y-10 scrollbar-hide">
              <div className="space-y-6">
                 <h2 className="text-[10px] font-black uppercase tracking-[0.4em] text-zinc-700 px-2 flex items-center justify-between">
                   Protocols
                   <Activity className="w-3.5 h-3.5" />
                 </h2>
                 <div className="space-y-1.5">
                   {LESSONS.map((lesson) => (
                     <div 
                       key={lesson.id}
                       onClick={() => { setActiveLesson(lesson); setLabMode(false); setIsSidebarOpen(false); }}
                       className={`group p-4 rounded-2xl cursor-pointer border transition-all duration-300 ${activeLesson?.id === lesson.id 
                         ? `bg-indigo-500/10 border-indigo-500/40 shadow-xl shadow-indigo-500/5` 
                         : 'bg-transparent border-transparent hover:bg-indigo-500/5 hover:border-indigo-500/10'}`}
                     >
                       <div className="text-[9px] font-black uppercase tracking-[0.2em] text-indigo-400/60 mb-1.5 group-hover:text-indigo-400 transition-colors">{lesson.category}</div>
                       <h3 className={`text-[13px] font-bold leading-tight tracking-tight ${activeLesson?.id === lesson.id ? 'text-white' : 'text-zinc-500 group-hover:text-zinc-300'}`}>
                         {lesson.title}
                       </h3>
                     </div>
                   ))}
                 </div>
              </div>

              <div className="space-y-6 pt-8 border-t border-white/5">
                 <h2 className="text-[10px] font-black uppercase tracking-[0.4em] text-zinc-700 px-2 flex items-center justify-between">
                   Neural Log
                   <History className="w-3.5 h-3.5 text-cyan-500/40" />
                 </h2>
                 <div className="space-y-2">
                   {chatHistory.filter(m => m.role === 'user').map((msg, i) => (
                     <div 
                       key={i}
                       onClick={() => scrollToMessage(chatHistory.indexOf(msg))}
                       className="group p-4 rounded-2xl cursor-pointer border border-transparent hover:bg-cyan-500/5 hover:border-cyan-500/20 transition-all flex flex-col gap-2"
                     >
                       <div className="flex items-center gap-3">
                          <Clock className="w-3 h-3 text-zinc-800 group-hover:text-cyan-600 transition-colors" />
                          <span className="text-[9px] font-black text-zinc-700 uppercase tracking-widest group-hover:text-cyan-500 transition-colors">
                            ARCHIVE_{i.toString().padStart(2, '0')}
                          </span>
                       </div>
                       <p className="text-[12px] text-zinc-600 truncate group-hover:text-zinc-300 font-medium">
                         {msg.text}
                       </p>
                     </div>
                   ))}
                 </div>
                 
                 <button 
                   onClick={clearHistory}
                   className="w-full mt-4 py-3 px-4 flex items-center justify-center gap-3 text-[10px] font-black text-rose-500/60 uppercase tracking-[0.3em] hover:text-rose-500 hover:bg-rose-500/5 rounded-xl transition-all border border-transparent hover:border-rose-500/20"
                 >
                   <Trash2 className="w-3.5 h-3.5" /> Purge Memory
                 </button>
              </div>

              <div className="space-y-6 pt-8 border-t border-white/5">
                 <h2 className="text-[10px] font-black uppercase tracking-[0.4em] text-rose-500/70 px-2 flex items-center justify-between">
                   Repository Sync
                   <GitBranch className="w-3.5 h-3.5" />
                 </h2>
                 <div className="p-5 glass-panel rounded-2xl border border-rose-500/20 space-y-4 relative group">
                    <div className="flex items-center justify-between">
                       <span className="text-[9px] font-black uppercase text-zinc-600 tracking-widest">Branch</span>
                       <span className="text-[10px] font-black text-rose-400 mono">MAIN // PROD</span>
                    </div>
                    <div className="flex items-center justify-between">
                       <span className="text-[9px] font-black uppercase text-zinc-600 tracking-widest">Hash</span>
                       <span className="text-[10px] font-black text-zinc-400 mono">{lastCommit}</span>
                    </div>
                    
                    {isSyncing ? (
                      <div className="space-y-2 animate-pulse">
                         <div className="h-1 bg-rose-500/20 rounded-full overflow-hidden">
                            <div className="h-full bg-rose-500 animate-[progress_2s_ease-in-out_infinite]" style={{ width: '40%' }}></div>
                         </div>
                         <div className="text-[8px] font-black text-rose-400 uppercase tracking-widest text-center">Neural Push in progress...</div>
                      </div>
                    ) : (
                      <button 
                        onClick={handleSync}
                        className="w-full py-2.5 bg-rose-600/10 hover:bg-rose-600 border border-rose-600/30 text-rose-400 hover:text-white rounded-xl text-[10px] font-black uppercase tracking-[0.2em] transition-all flex items-center justify-center gap-2 group-hover:shadow-[0_0_15px_rgba(244,63,94,0.3)]"
                      >
                        <CloudUpload className="w-3.5 h-3.5" /> Initiate Push
                      </button>
                    )}
                 </div>
                 <button 
                   onClick={() => setShowGuide(true)}
                   className="px-2 flex items-center gap-2 text-[8px] font-black text-zinc-600 uppercase tracking-[0.3em] hover:text-rose-400 transition-colors"
                 >
                    <Book className="w-3 h-3" /> Deployment Guide
                 </button>
                 <div className="px-2 flex items-center gap-2">
                    <CheckCircle2 className="w-3 h-3 text-emerald-500" />
                    <span className="text-[8px] font-black text-zinc-700 uppercase tracking-[0.4em]">Web Live Status: Active</span>
                 </div>
              </div>
           </div>

           <div className="p-6 border-t border-white/5 bg-black/50 backdrop-blur-md">
              <PhonkFM />
           </div>
        </aside>

        <div className="flex-1 flex flex-col relative overflow-hidden bg-black">
          <div ref={scrollContainerRef} className="flex-1 overflow-y-auto p-4 md:p-12 pb-40 scroll-smooth custom-scrollbar">
             <div className="max-w-4xl mx-auto w-full space-y-12">
                {permissionError && (
                   <div className="p-5 bg-rose-500/10 border border-rose-500/30 rounded-[1.5rem] text-rose-400 text-xs font-black uppercase tracking-widest flex items-center gap-4 animate-in slide-in-from-top-2">
                      <AlertTriangle className="w-5 h-5 flex-shrink-0 animate-pulse" />
                      <span>System Override: {permissionError}</span>
                   </div>
                )}

                {isLiveActive ? (
                   <div className="h-full min-h-[60vh] flex flex-col items-center justify-center space-y-12 animate-in zoom-in-95 duration-700">
                      <LiveVoice isActive={isLiveActive} isSpeaking={isSpeaking} transcript={liveTranscript} />
                      <div className="text-center space-y-6">
                         <h2 className="text-4xl font-black text-white uppercase tracking-tighter text-glow-indigo">Neural Link Active</h2>
                         <button onClick={toggleLive} className="px-12 py-4 bg-rose-600 hover:bg-rose-500 text-white font-black uppercase text-[11px] tracking-[0.4em] rounded-[1.5rem] transition-all shadow-[0_0_30px_rgba(225,29,72,0.3)]">Disconnect Link</button>
                      </div>
                   </div>
                ) : labMode ? (
                   <div className="animate-in fade-in slide-in-from-bottom-10 duration-700">
                      <div className="flex items-center justify-between mb-8">
                        <button onClick={() => setLabMode(false)} className="px-6 py-3 glass-panel border border-white/10 rounded-2xl text-zinc-400 text-[10px] font-black uppercase tracking-[0.3em] flex items-center gap-3 hover:text-white transition-all hover:border-white/20">
                          <ChevronRight className="w-4 h-4 rotate-180" /> Termination Sequence
                        </button>
                        <div className="text-[11px] font-black uppercase tracking-[0.4em] text-indigo-400 flex items-center gap-3">
                          <Zap className="w-5 h-5 animate-pulse cyber-glow" /> Simulation Running
                        </div>
                      </div>
                      <div className="p-1 rounded-[2.5rem] bg-gradient-to-br from-indigo-500/20 to-transparent">
                        {renderLab()}
                      </div>
                   </div>
                ) : (
                   <div className="space-y-16 pb-12">
                      {activeLesson && (
                        <div className="p-10 rounded-[3rem] glass-panel border border-indigo-500/20 space-y-8 animate-in fade-in zoom-in-95 duration-500 relative group overflow-hidden">
                           <div className="absolute inset-0 bg-indigo-500/5 opacity-0 group-hover:opacity-100 transition-opacity duration-1000"></div>
                           <div className="absolute top-6 right-8 text-[9px] font-black text-indigo-500 uppercase tracking-[0.5em] animate-pulse">Synchronized</div>
                           <div className="w-16 h-16 rounded-[1.5rem] bg-indigo-500/10 border border-indigo-500/30 flex items-center justify-center text-indigo-400 cyber-glow shadow-[0_0_20px_rgba(99,102,241,0.2)]">
                              <Zap className="w-8 h-8" />
                           </div>
                           <div className="space-y-4 relative">
                              <h1 className="text-4xl font-black text-white uppercase tracking-tighter leading-none">{activeLesson.title}</h1>
                              <p className="text-zinc-400 text-[15px] leading-relaxed max-w-2xl font-medium">{activeLesson.content}</p>
                           </div>
                           {activeLesson.hasLab && (
                              <button onClick={() => setLabMode(true)} className="px-10 py-4 bg-indigo-600 hover:bg-indigo-500 text-white font-black uppercase tracking-[0.3em] text-[11px] rounded-[1.5rem] shadow-[0_0_30px_rgba(99,102,241,0.4)] transition-all hover:scale-105 active:scale-95">
                                Deploy Simulation
                              </button>
                           )}
                        </div>
                      )}

                      <div className="space-y-12">
                         {chatHistory.map((msg, i) => (
                            <div 
                              key={i} 
                              ref={el => { messageRefs.current[i] = el; }}
                              className={`flex flex-col ${msg.role === 'user' ? 'items-end' : 'items-start'} animate-in fade-in slide-in-from-bottom-5 duration-500`}
                            >
                               <div className={`max-w-[85%] p-7 rounded-[2rem] text-[15px] leading-relaxed shadow-2xl transition-all ${
                                  msg.role === 'user' 
                                  ? 'bg-white text-black font-bold shadow-white/5 border border-white' 
                                  : 'bg-zinc-900/40 backdrop-blur-xl border border-white/5 text-zinc-100 font-medium'
                               }`}>
                                  <MessageContent text={msg.text} />
                                  
                                  {msg.sources && msg.sources.length > 0 && (
                                     <div className="mt-8 pt-6 border-t border-white/5 flex flex-wrap gap-3">
                                        {msg.sources.map((s, si) => (
                                           <a key={si} href={s.uri} target="_blank" className="px-4 py-2 rounded-xl bg-indigo-500/10 text-indigo-300 border border-indigo-500/20 text-[10px] font-black uppercase hover:bg-indigo-500/20 transition-all flex items-center gap-3">
                                              <Globe className="w-3.5 h-3.5" /> INTEL_{si+1}
                                           </a>
                                        ))}
                                     </div>
                                  )}
                               </div>
                            </div>
                         ))}
                         <div ref={chatEndRef} />
                      </div>
                   </div>
                )}
             </div>
          </div>

          {!labMode && !isLiveActive && (
             <div className="fixed bottom-10 left-1/2 -translate-x-1/2 w-full max-w-3xl px-6 z-[120]">
                <form 
                   onSubmit={handleSendMessage} 
                   className={`relative group bg-black/80 backdrop-blur-3xl border border-white/5 rounded-[3rem] p-2.5 shadow-[0_0_50px_rgba(0,0,0,0.8)] transition-all duration-500 ${isTyping ? 'ring-2 ring-indigo-500/50' : 'hover:border-white/20'}`}
                >
                   <div className="absolute inset-x-0 -bottom-1 h-px bg-gradient-to-r from-transparent via-indigo-500/50 to-transparent blur-sm"></div>
                   <div className="flex items-center gap-4 px-6 py-4">
                      <div className="p-3 rounded-2xl bg-indigo-500/10 text-indigo-400 shadow-[0_0_15px_rgba(99,102,241,0.2)]">
                         <MessageSquare className="w-6 h-6" />
                      </div>
                      <input 
                         type="text"
                         value={userInput}
                         onChange={(e) => setUserInput(e.target.value)}
                         placeholder="Transmit neural directive, Commander..."
                         className="flex-1 bg-transparent border-none outline-none text-white text-[16px] font-bold placeholder:text-zinc-700 tracking-tight"
                         disabled={isTyping}
                      />
                      <button 
                         type="submit" 
                         disabled={!userInput.trim() || isTyping}
                         className={`p-5 rounded-[2rem] bg-indigo-600 text-white shadow-2xl transition-all active:scale-90 disabled:opacity-20 disabled:grayscale hover:bg-indigo-500 hover:shadow-[0_0_25px_rgba(99,102,241,0.6)]`}
                      >
                         {isTyping ? <RefreshCw className="w-6 h-6 animate-spin" /> : <Send className="w-6 h-6" />}
                      </button>
                   </div>
                </form>
             </div>
          )}
        </div>
      </main>
    </div>
  );
};

export default App;
