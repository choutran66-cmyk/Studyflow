# Studyflow
import React, { useState, useEffect } from 'react';
import { motion, AnimatePresence } from 'motion/react';
import { 
  LayoutDashboard, 
  Calendar, 
  Timer, 
  BarChart2, 
  Sparkles,
  LogOut,
  Menu,
  X
} from 'lucide-react';
import AssessmentForm from './components/AssessmentForm';
import Schedule from './components/Schedule';
import FocusTimer from './components/FocusTimer';
import Analytics from './components/Analytics';
import { User, Task, AnalyticsData } from './types';
import { generateStudyPlan, getMotivationalMessage } from './services/gemini';

export default function App() {
  const [user, setUser] = useState<User | null>(null);
  const [tasks, setTasks] = useState<Task[]>([]);
  const [analytics, setAnalytics] = useState<AnalyticsData | null>(null);
  const [activeTab, setActiveTab] = useState<'dashboard' | 'schedule' | 'focus' | 'analytics'>('dashboard');
  const [selectedTask, setSelectedTask] = useState<Task | null>(null);
  const [motivation, setMotivation] = useState<string>('');
  const [loading, setLoading] = useState(true);
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);

  useEffect(() => {
    fetchInitialData();
  }, []);

  const fetchInitialData = async () => {
    try {
      const userRes = await fetch('/api/user');
      const userData = await userRes.json();
      
      if (userData) {
        setUser(userData);
        await fetchTasks();
        await fetchAnalytics();
        const msg = await getMotivationalMessage('Welcome back! Ready to crush your goals?');
        setMotivation(msg);
      }
    } catch (error) {
      console.error('Error fetching data:', error);
    } finally {
      setLoading(false);
    }
  };

  const fetchTasks = async () => {
    const res = await fetch('/api/tasks');
    const data = await res.json();
    setTasks(data);
    if (data.length > 0 && !selectedTask) {
      setSelectedTask(data.find((t: Task) => t.status === 'pending') || data[0]);
    }
  };

  const fetchAnalytics = async () => {
    const res = await fetch('/api/analytics');
    const data = await res.json();
    setAnalytics(data);
  };

  const handleAssessmentSubmit = async (data: any) => {
    setLoading(true);
    try {
      const userRes = await fetch('/api/user', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });
      const userData = await userRes.json();
      const newUser = { ...data, id: userData.id };
      setUser(newUser);

      const plan = await generateStudyPlan(newUser);
      for (const task of plan) {
        await fetch('/api/tasks', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ ...task, user_id: userData.id }),
        });
      }
      await fetchTasks();
      await fetchAnalytics();
      setActiveTab('dashboard');
    } catch (error) {
      console.error('Error generating plan:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleToggleStatus = async (id: number, status: Task['status']) => {
    await fetch(`/api/tasks/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ status }),
    });
    await fetchTasks();
    await fetchAnalytics();
    
    if (status === 'completed') {
      const msg = await getMotivationalMessage('Completed a task! Great momentum.');
      setMotivation(msg);
    }
  };

  const handleFocusComplete = async (duration: number, score: number) => {
    if (!user || !selectedTask) return;
    
    await fetch('/api/focus-sessions', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        user_id: user.id,
        task_id: selectedTask.id,
        duration,
        focus_score: score,
      }),
    });
    
    await fetchAnalytics();
    const msg = await getMotivationalMessage(`Finished a ${duration}m focus session with score ${score}/5.`);
    setMotivation(msg);
  };

  if (loading) {
    return (
      <div className="min-h-screen bg-zinc-50 flex items-center justify-center">
        <div className="flex flex-col items-center gap-4">
          <div className="w-12 h-12 border-4 border-emerald-500 border-t-transparent rounded-full animate-spin" />
          <p className="text-zinc-500 font-medium animate-pulse">Syncing with StudyFlow AI...</p>
        </div>
      </div>
    );
  }

  if (!user) {
    return (
      <div className="min-h-screen bg-zinc-50 p-6 flex items-center justify-center">
        <AssessmentForm onSubmit={handleAssessmentSubmit} />
      </div>
    );
  }

  const navItems = [
    { id: 'dashboard', label: 'Dashboard', icon: LayoutDashboard },
    { id: 'schedule', label: 'Schedule', icon: Calendar },
    { id: 'focus', label: 'Focus Room', icon: Timer },
    { id: 'analytics', label: 'Analytics', icon: BarChart2 },
  ];

  return (
    <div className="min-h-screen bg-zinc-50 flex">
      {/* Sidebar - Desktop */}
      <aside className="hidden lg:flex flex-col w-72 bg-white border-r border-zinc-200 p-6">
        <div className="flex items-center gap-3 mb-10 px-2">
          <div className="w-10 h-10 bg-emerald-500 rounded-xl flex items-center justify-center text-white shadow-lg shadow-emerald-500/20">
            <Sparkles size={24} />
          </div>
          <h1 className="text-xl font-bold tracking-tight text-zinc-900">StudyFlow AI</h1>
        </div>

        <nav className="flex-1 space-y-2">
          {navItems.map((item) => (
            <button
              key={item.id}
              onClick={() => setActiveTab(item.id as any)}
              className={`w-full flex items-center gap-3 px-4 py-3 rounded-xl font-medium transition-all ${
                activeTab === item.id
                  ? 'bg-emerald-50 text-emerald-600'
                  : 'text-zinc-500 hover:bg-zinc-50 hover:text-zinc-900'
              }`}
            >
              <item.icon size={20} />
              {item.label}
            </button>
          ))}
        </nav>

        <div className="mt-auto pt-6 border-t border-zinc-100">
          <div className="px-4 py-3 bg-zinc-900 rounded-2xl text-white mb-4">
            <p className="text-[10px] uppercase tracking-widest opacity-50 mb-1 font-bold">Daily Motivation</p>
            <p className="text-sm italic font-serif leading-relaxed">"{motivation}"</p>
          </div>
          <button className="w-full flex items-center gap-3 px-4 py-3 text-zinc-500 hover:text-red-500 transition-colors font-medium">
            <LogOut size={20} />
            Sign Out
          </button>
        </div>
      </aside>

      {/* Mobile Header */}
      <div className="lg:hidden fixed top-0 left-0 right-0 bg-white border-b border-zinc-200 z-50 px-6 py-4 flex items-center justify-between">
        <div className="flex items-center gap-2">
          <Sparkles className="text-emerald-500" size={24} />
          <span className="font-bold text-zinc-900">StudyFlow AI</span>
        </div>
        <button onClick={() => setIsSidebarOpen(!isSidebarOpen)}>
          {isSidebarOpen ? <X /> : <Menu />}
        </button>
      </div>

      {/* Mobile Sidebar Overlay */}
      <AnimatePresence>
        {isSidebarOpen && (
          <>
            <motion.div
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              exit={{ opacity: 0 }}
              onClick={() => setIsSidebarOpen(false)}
              className="fixed inset-0 bg-black/20 backdrop-blur-sm z-40 lg:hidden"
            />
            <motion.aside
              initial={{ x: '-100%' }}
              animate={{ x: 0 }}
              exit={{ x: '-100%' }}
              className="fixed top-0 left-0 bottom-0 w-72 bg-white z-50 p-6 lg:hidden"
            >
              <div className="flex items-center gap-3 mb-10">
                <Sparkles className="text-emerald-500" size={24} />
                <h1 className="text-xl font-bold text-zinc-900">StudyFlow AI</h1>
              </div>
              <nav className="space-y-2">
                {navItems.map((item) => (
                  <button
                    key={item.id}
                    onClick={() => {
                      setActiveTab(item.id as any);
                      setIsSidebarOpen(false);
                    }}
                    className={`w-full flex items-center gap-3 px-4 py-3 rounded-xl font-medium ${
                      activeTab === item.id ? 'bg-emerald-50 text-emerald-600' : 'text-zinc-500'
                    }`}
                  >
                    <item.icon size={20} />
                    {item.label}
                  </button>
                ))}
              </nav>
            </motion.aside>
          </>
        )}
      </AnimatePresence>

      {/* Main Content */}
      <main className="flex-1 p-6 lg:p-10 mt-16 lg:mt-0 overflow-y-auto">
        <div className="max-w-5xl mx-auto">
          <header className="mb-10">
            <h2 className="text-3xl font-bold text-zinc-900 mb-2">
              {activeTab === 'dashboard' && `Welcome back, ${user.name}`}
              {activeTab === 'schedule' && 'Your Study Schedule'}
              {activeTab === 'focus' && 'Deep Focus Room'}
              {activeTab === 'analytics' && 'Performance Insights'}
            </h2>
            <p className="text-zinc-500">
              {activeTab === 'dashboard' && "Here's what's on your plate for today."}
              {activeTab === 'schedule' && "A balanced plan crafted by AI for your goals."}
              {activeTab === 'focus' && "Minimize distractions and maximize your learning."}
              {activeTab === 'analytics' && "Data-driven insights into your study habits."}
            </p>
          </header>

          <AnimatePresence mode="wait">
            <motion.div
              key={activeTab}
              initial={{ opacity: 0, y: 10 }}
              animate={{ opacity: 1, y: 0 }}
              exit={{ opacity: 0, y: -10 }}
              transition={{ duration: 0.2 }}
            >
              {activeTab === 'dashboard' && (
                <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
                  <div className="lg:col-span-2 space-y-8">
                    <section>
                      <div className="flex items-center justify-between mb-4">
                        <h3 className="font-bold text-zinc-900">Priority Tasks</h3>
                        <button onClick={() => setActiveTab('schedule')} className="text-sm text-emerald-600 font-medium hover:underline">View All</button>
                      </div>
                      <Schedule 
                        tasks={tasks.filter(t => t.status === 'pending').slice(0, 3)} 
                        onToggleStatus={handleToggleStatus}
                        onSelectTask={(task) => {
                          setSelectedTask(task);
                          setActiveTab('focus');
                        }}
                      />
                    </section>
                    
                    {analytics && (
                      <section>
                        <h3 className="font-bold text-zinc-900 mb-4">Weekly Progress</h3>
                        <Analytics data={analytics} />
                      </section>
                    )}
                  </div>

                  <div className="space-y-8">
                    <div className="bg-white p-6 rounded-3xl border border-zinc-100 shadow-sm">
                      <h3 className="font-bold text-zinc-900 mb-4">Current Focus</h3>
                      {selectedTask ? (
                        <div className="space-y-4">
                          <div className="p-4 bg-zinc-50 rounded-2xl">
                            <p className="text-xs font-bold text-zinc-400 uppercase tracking-widest mb-1">{selectedTask.subject}</p>
                            <p className="font-medium text-zinc-900">{selectedTask.title}</p>
                          </div>
                          <button 
                            onClick={() => setActiveTab('focus')}
                            className="w-full py-3 bg-emerald-500 text-white rounded-xl font-semibold hover:bg-emerald-600 transition-colors shadow-lg shadow-emerald-500/20"
                          >
                            Start Session
                          </button>
                        </div>
                      ) : (
                        <p className="text-zinc-500 text-sm italic">No task selected. Pick one from your schedule!</p>
                      )}
                    </div>

                    <div className="bg-emerald-600 p-6 rounded-3xl text-white shadow-xl shadow-emerald-600/20">
                      <Sparkles className="mb-4" size={24} />
                      <h3 className="font-bold text-lg mb-2">AI Tip</h3>
                      <p className="text-emerald-100 text-sm leading-relaxed">
                        "Try alternating between a high-intensity subject like Math and a creative one like Literature to keep your brain fresh."
                      </p>
                    </div>
                  </div>
                </div>
              )}

              {activeTab === 'schedule' && (
                <div className="max-w-2xl">
                  <Schedule 
                    tasks={tasks} 
                    onToggleStatus={handleToggleStatus}
                    onSelectTask={setSelectedTask}
                    selectedTaskId={selectedTask?.id}
                  />
                </div>
              )}

              {activeTab === 'focus' && (
                <div className="max-w-xl mx-auto">
                  <FocusTimer 
                    taskId={selectedTask?.id} 
                    taskTitle={selectedTask?.title}
                    onComplete={handleFocusComplete}
                  />
                  <div className="mt-8 p-6 bg-white rounded-3xl border border-zinc-100 shadow-sm">
                    <h4 className="font-bold text-zinc-900 mb-4">Focus Tips</h4>
                    <ul className="space-y-3 text-sm text-zinc-600">
                      <li className="flex gap-2">
                        <div className="w-1.5 h-1.5 rounded-full bg-emerald-500 mt-1.5 shrink-0" />
                        Put your phone in another room to minimize distractions.
                      </li>
                      <li className="flex gap-2">
                        <div className="w-1.5 h-1.5 rounded-full bg-emerald-500 mt-1.5 shrink-0" />
                        Keep a glass of water nearby to stay hydrated.
                      </li>
                      <li className="flex gap-2">
                        <div className="w-1.5 h-1.5 rounded-full bg-emerald-500 mt-1.5 shrink-0" />
                        Use noise-canceling headphones or lo-fi music.
                      </li>
                    </ul>
                  </div>
                </div>
              )}

              {activeTab === 'analytics' && analytics && (
                <Analytics data={analytics} />
              )}
            </motion.div>
          </AnimatePresence>
        </div>
      </main>
    </div>
  );
}

