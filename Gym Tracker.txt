import React, { useState, useMemo, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, onSnapshot, setDoc } from 'firebase/firestore';
import { 
  Calendar as CalendarIcon, 
  CalendarDays,
  BarChart2, 
  Settings, 
  Plus, 
  ChevronLeft, 
  ChevronRight, 
  CheckCircle2, 
  Circle, 
  Trash2, 
  Edit2, 
  Moon, 
  Sun, 
  Dumbbell,
  History,
  TrendingUp,
  X,
  Save,
  LayoutGrid,
  PlusCircle,
  ChevronDown,
  CalendarDays as CalendarEdit,
  Loader2,
  Weight,
  Filter,
  Check,
  GripVertical,
  Activity
} from 'lucide-react';
import { 
  BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, 
  LineChart, Line 
} from 'recharts';

// --- CONFIGURATION ---
const firebaseConfig = {
  apiKey: "AIzaSyA4gVc2E8w5-Juc14g_wgwLrGK_NRfDkBc",
  authDomain: "gym-tracker-7cc5e.firebaseapp.com",
  projectId: "gym-tracker-7cc5e",
  storageBucket: "gym-tracker-7cc5e.firebasestorage.app",
  messagingSenderId: "1091384077962",
  appId: "1:1091384077962:web:db39542aeb62ce92f80773",
  measurementId: "G-GR1S5KX05F"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const DB_APP_ID = "gym-tracker-7cc5e";

const EQUIPMENT_TYPES = [
  'Собственный вес',
  'Штанга',
  'Смит',
  'Гантели',
  'Тренажёр',
  'Кроссовер',
  'Гири'
];

const INITIAL_EXERCISES = [
  { id: '1', name: 'Жим лежа', category: 'Грудь', equipment: 'Штанга' },
  { id: '2', name: 'Приседания', category: 'Ноги', equipment: 'Штанга' },
  { id: '3', name: 'Становая тяга', category: 'Спина', equipment: 'Штанга' },
  { id: '4', name: 'Подтягивания', category: 'Спина', equipment: 'Собственный вес' },
  { id: '5', name: 'Армейский жим', category: 'Плечи', equipment: 'Штанга' },
];

const MUSCLE_GROUPS = ['Грудь', 'Спина', 'Ноги', 'Плечи', 'Руки', 'Пресс', 'Кардио', 'Другое'];

const formatDate = (date) => {
  const d = new Date(date);
  return d.toISOString().split('T')[0];
};

const getMonthDays = (year, month) => {
  const date = new Date(year, month, 1);
  const days = [];
  while (date.getMonth() === month) {
    days.push(new Date(date));
    date.setDate(date.getDate() + 1);
  }
  return days;
};

const sortExercises = (list, mode) => {
  return [...list].sort((a, b) => {
    if (mode === 'muscle') {
      return a.category.localeCompare(b.category, 'ru') || a.name.localeCompare(b.name, 'ru');
    }
    if (mode === 'equipment') {
      return (a.equipment || '').localeCompare(b.equipment || '', 'ru') || a.name.localeCompare(b.name, 'ru');
    }
    return a.name.localeCompare(b.name, 'ru');
  });
};

// --- Custom UI Components ---

function CustomSelect({ options, value, onChange, label }) {
  const [isOpen, setIsOpen] = useState(false);
  const dropdownRef = useRef(null);

  useEffect(() => {
    const handleClickOutside = (event) => {
      if (dropdownRef.current && !dropdownRef.current.contains(event.target)) setIsOpen(false);
    };
    document.addEventListener('mousedown', handleClickOutside);
    return () => document.removeEventListener('mousedown', handleClickOutside);
  }, []);

  const selectedOption = options.find(opt => opt.value === value);

  return (
    <div className="relative" ref={dropdownRef}>
      <button 
        onClick={() => setIsOpen(!isOpen)}
        className="flex items-center gap-2 bg-white dark:bg-gray-800 px-4 py-2 rounded-2xl border border-gray-100 dark:border-gray-700 shadow-sm transition-all active:scale-95"
      >
        <Filter size={14} className="text-blue-500" />
        <span className="text-[11px] font-black uppercase text-gray-500 dark:text-gray-400">
          {label}: {selectedOption?.label}
        </span>
        <ChevronDown size={14} className={`text-gray-400 transition-transform ${isOpen ? 'rotate-180' : ''}`} />
      </button>

      {isOpen && (
        <div className="absolute right-0 mt-2 w-48 bg-white dark:bg-gray-800 rounded-2xl border border-gray-100 dark:border-gray-700 shadow-xl z-[100] py-2 animate-in fade-in zoom-in-95 duration-100">
          {options.map((opt) => (
            <button
              key={opt.value}
              onClick={() => { onChange(opt.value); setIsOpen(false); }}
              className="w-full flex items-center justify-between px-4 py-3 text-left hover:bg-gray-50 dark:hover:bg-gray-700 transition-colors"
            >
              <span className={`text-xs font-black uppercase ${value === opt.value ? 'text-blue-500' : 'text-gray-500 dark:text-gray-400'}`}>{opt.label}</span>
              {value === opt.value && <Check size={14} className="text-blue-500" />}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}

// --- MAIN APP ---

export default function App() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [isSyncing, setIsSyncing] = useState(false);
  
  const [activeTab, setActiveTab] = useState('workouts');
  const [theme, setTheme] = useState('dark');
  const [units, setUnits] = useState('kg');
  const [sortMode, setSortMode] = useState('name');
  const [exercises, setExercises] = useState(INITIAL_EXERCISES);
  const [workouts, setWorkouts] = useState([]);
  const [templates, setTemplates] = useState([]);
  
  const [selectedDate, setSelectedDate] = useState(formatDate(new Date()));
  const [isCalendarExpanded, setIsCalendarExpanded] = useState(false);
  const [editingWorkout, setEditingWorkout] = useState(null);
  const [editingTemplate, setEditingTemplate] = useState(null);
  const [editingExercise, setEditingExercise] = useState(null);
  const [selectedExerciseStats, setSelectedExerciseStats] = useState(null);

  useEffect(() => {
    const login = async () => {
      try { await signInAnonymously(auth); } catch (err) { console.error("Auth error:", err); }
    };
    login();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    const dataDocRef = doc(db, 'artifacts', DB_APP_ID, 'public', 'data', 'user_state', 'main');
    const unsub = onSnapshot(dataDocRef, (docSnap) => {
      if (docSnap.exists()) {
        const data = docSnap.data();
        if (data.workouts) setWorkouts(data.workouts);
        if (data.exercises) setExercises(data.exercises);
        if (data.templates) setTemplates(data.templates);
        if (data.units) setUnits(data.units);
        if (data.theme) setTheme(data.theme);
        if (data.sortMode) setSortMode(data.sortMode);
      } else {
        setDoc(dataDocRef, { workouts: [], exercises: INITIAL_EXERCISES, templates: [], units: 'kg', theme: 'dark', sortMode: 'name' });
      }
      setLoading(false);
    }, (err) => {
      console.error("Firestore sync error:", err);
      setLoading(false);
    });
    return () => unsub();
  }, [user]);

  useEffect(() => {
    document.documentElement.classList.toggle('dark', theme === 'dark');
  }, [theme]);

  const syncWithFirebase = async (payload) => {
    if (!user) return;
    setIsSyncing(true);
    try {
      const dataDocRef = doc(db, 'artifacts', DB_APP_ID, 'public', 'data', 'user_state', 'main');
      await setDoc(dataDocRef, payload, { merge: true });
    } catch (err) {
      console.error("Update error:", err);
    } finally {
      setIsSyncing(false);
    }
  };

  const totalTonnage = useMemo(() => {
    return workouts.reduce((sum, w) => sum + w.sets.reduce((sSum, s) => s.completed ? sSum + (Number(s.weight) * Number(s.reps)) : sSum, 0), 0);
  }, [workouts]);

  const activityData = useMemo(() => {
    const months = [];
    for (let i = 5; i >= 0; i--) {
      const d = new Date(); d.setMonth(d.getMonth() - i);
      const label = d.toLocaleString('default', { month: 'short' });
      const count = workouts.filter(w => {
        const wd = new Date(w.date);
        return wd.getMonth() === d.getMonth() && wd.getFullYear() === d.getFullYear();
      }).length;
      months.push({ name: label, count });
    }
    return months;
  }, [workouts]);

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-900 text-white">
        <div className="flex flex-col items-center gap-4 text-center px-6">
          <Dumbbell size={64} className="animate-spin text-blue-500 mb-2" />
          <h2 className="text-xl font-black uppercase tracking-tighter">Gym Tracker</h2>
          <p className="text-xs text-gray-500 font-bold uppercase tracking-widest animate-pulse">Синхронизация...</p>
        </div>
      </div>
    );
  }

  return (
    <div className={`min-h-screen font-sans ${theme === 'dark' ? 'bg-gray-900 text-white' : 'bg-gray-50 text-gray-900'}`}>
      <style dangerouslySetInnerHTML={{ __html: `
        input::-webkit-outer-spin-button, input::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }
        input[type=number] { -moz-appearance: textfield; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
      `}} />

      {isSyncing && (
        <div className="fixed top-4 right-4 z-[100] bg-blue-500 text-white p-2 rounded-full shadow-lg">
          <Loader2 size={16} className="animate-spin" />
        </div>
      )}

      <aside className="hidden md:fixed md:flex flex-col w-72 h-full bg-white dark:bg-gray-800 border-r dark:border-gray-700 p-8 z-40 shadow-sm">
        <div className="flex items-center gap-4 mb-14">
          <div className="w-12 h-12 bg-blue-500 rounded-2xl flex items-center justify-center text-white shadow-sm shadow-blue-200 dark:shadow-none"><Dumbbell size={24} /></div>
          <h1 className="text-2xl font-black tracking-tighter dark:text-white uppercase">GYM<span className="text-blue-500">TRACKER</span></h1>
        </div>
        <nav className="space-y-3 flex-1">
          <button onClick={() => setActiveTab('workouts')} className={`w-full flex items-center gap-4 p-5 rounded-3xl font-black text-xs uppercase tracking-widest transition-all ${activeTab === 'workouts' ? 'bg-blue-500 text-white shadow-sm' : 'text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'}`}><CalendarIcon size={18} /> Тренировки</button>
          <button onClick={() => setActiveTab('stats')} className={`w-full flex items-center gap-4 p-5 rounded-3xl font-black text-xs uppercase tracking-widest transition-all ${activeTab === 'stats' ? 'bg-blue-500 text-white shadow-sm' : 'text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'}`}><BarChart2 size={18} /> Статистика</button>
          <button onClick={() => setActiveTab('settings')} className={`w-full flex items-center gap-4 p-5 rounded-3xl font-black text-xs uppercase tracking-widest transition-all ${activeTab === 'settings' ? 'bg-blue-500 text-white shadow-sm' : 'text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'}`}><Settings size={18} /> Настройки</button>
        </nav>
      </aside>

      <main className="md:ml-72 p-5 md:p-14 max-w-5xl mx-auto min-h-screen transition-colors">
        <header className="md:hidden flex items-center justify-between mb-8 pt-4">
          <h1 className="text-3xl font-black tracking-tighter dark:text-white uppercase">{activeTab === 'workouts' ? 'Тренировки' : activeTab === 'stats' ? 'Статистика' : 'Настройки'}</h1>
          <div className="w-12 h-12 bg-blue-500 rounded-2xl flex items-center justify-center text-white shadow-sm transition-transform active:scale-95"><Dumbbell size={24} /></div>
        </header>

        {activeTab === 'workouts' && (
          <WorkoutsView 
            workouts={workouts} exercises={exercises} units={units} 
            selectedDate={selectedDate} setSelectedDate={setSelectedDate}
            isCalendarExpanded={isCalendarExpanded} setIsCalendarExpanded={setIsCalendarExpanded}
            setEditingWorkout={setEditingWorkout} syncWithFirebase={syncWithFirebase}
            templates={templates}
          />
        )}
        {activeTab === 'stats' && <StatsView workouts={workouts} exercises={exercises} activityData={activityData} units={units} setSelectedExerciseStats={setSelectedExerciseStats} sortMode={sortMode} setSortMode={(m) => {setSortMode(m); syncWithFirebase({sortMode: m});}} />}
        {activeTab === 'settings' && <SettingsView theme={theme} units={units} exercises={exercises} templates={templates} setEditingTemplate={setEditingTemplate} setEditingExercise={setEditingExercise} syncWithFirebase={syncWithFirebase} sortMode={sortMode} setSortMode={(m) => {setSortMode(m); syncWithFirebase({sortMode: m});}} />}
      </main>

      <nav className="md:hidden fixed bottom-0 left-0 right-0 bg-white/95 dark:bg-gray-800/95 backdrop-blur-2xl border-t border-gray-100 dark:border-gray-700 px-6 py-4 pb-10 z-40 flex justify-center gap-12 items-center">
        <button onClick={() => setActiveTab('stats')} className={`flex flex-col items-center gap-2 transition-all ${activeTab === 'stats' ? 'text-blue-500 scale-110' : 'text-gray-300'}`}><BarChart2 size={28} /><span className="text-[10px] font-black uppercase tracking-tight">Статистика</span></button>
        <button onClick={() => setActiveTab('workouts')} className={`flex flex-col items-center gap-2 transition-all ${activeTab === 'workouts' ? 'text-blue-500 scale-110' : 'text-gray-300'}`}><CalendarIcon size={28} /><span className="text-[10px] font-black uppercase tracking-tight">Тренировки</span></button>
        <button onClick={() => setActiveTab('settings')} className={`flex flex-col items-center gap-2 transition-all ${activeTab === 'settings' ? 'text-blue-500 scale-110' : 'text-gray-300'}`}><Settings size={28} /><span className="text-[10px] font-black uppercase tracking-tight">Настройки</span></button>
      </nav>

      <WorkoutEditor 
        editingWorkout={editingWorkout} setEditingWorkout={setEditingWorkout}
        workouts={workouts} templates={templates} exercises={exercises} units={units}
        syncWithFirebase={syncWithFirebase} setSelectedDate={setSelectedDate}
      />
      <TemplateEditorModal 
        editingTemplate={editingTemplate} setEditingTemplate={setEditingTemplate}
        templates={templates} exercises={exercises} syncWithFirebase={syncWithFirebase}
        sortMode={sortMode} setSortMode={(m) => {setSortMode(m); syncWithFirebase({sortMode: m});}}
      />
      <ExerciseEditorModal 
        editingExercise={editingExercise} setEditingExercise={setEditingExercise}
        exercises={exercises} syncWithFirebase={syncWithFirebase}
      />
      <ExerciseStatsModal 
        selectedExerciseStats={selectedExerciseStats} setSelectedExerciseStats={setSelectedExerciseStats}
        workouts={workouts} units={units} exercises={exercises}
      />
    </div>
  );
}

// --- Grouping Component ---

function GroupedExerciseList({ exercises, sortMode, onItemClick, renderRightElement }) {
  const groups = useMemo(() => {
    const sorted = [...exercises].sort((a, b) => {
      if (sortMode === 'muscle') return a.category.localeCompare(b.category, 'ru') || a.name.localeCompare(b.name, 'ru');
      if (sortMode === 'equipment') return (a.equipment || '').localeCompare(b.equipment || '', 'ru') || a.name.localeCompare(b.name, 'ru');
      return a.name.localeCompare(b.name, 'ru');
    });

    const grouped = {};
    if (sortMode === 'name') {
      grouped['Все упражнения'] = sorted;
    } else {
      sorted.forEach(ex => {
        const key = sortMode === 'muscle' ? ex.category : (ex.equipment || 'Другое');
        if (!grouped[key]) grouped[key] = [];
        grouped[key].push(ex);
      });
    }
    return grouped;
  }, [exercises, sortMode]);

  return (
    <div className="space-y-6">
      {Object.entries(groups).map(([groupTitle, list]) => (
        <div key={groupTitle} className="space-y-3">
          {sortMode !== 'name' && (
            <div className="flex items-center gap-3 ml-1">
              <div className="h-px flex-1 bg-gray-100 dark:bg-gray-800" />
              <h4 className="text-[10px] font-black uppercase text-gray-400 tracking-widest px-2">{groupTitle}</h4>
              <div className="h-px flex-1 bg-gray-100 dark:bg-gray-800" />
            </div>
          )}
          <div className="grid gap-2">
            {list.map(ex => (
              <div 
                key={ex.id} 
                onClick={() => onItemClick && onItemClick(ex)}
                className={`flex items-center justify-between p-4 bg-white dark:bg-gray-800 rounded-3xl shadow-sm border border-gray-50 dark:border-gray-700/50 transition-all ${onItemClick ? 'active:scale-95 cursor-pointer hover:border-blue-500/30' : ''}`}
              >
                <div>
                  <p className="text-sm font-black dark:text-white uppercase tracking-tight">{ex.name}</p>
                  <div className="flex gap-2 mt-0.5">
                    <span className="text-[9px] font-black uppercase text-blue-500">{ex.category}</span>
                    {ex.equipment && <span className="text-[9px] font-black uppercase text-gray-400 border border-gray-100 dark:border-gray-700 px-1.5 rounded">{ex.equipment}</span>}
                  </div>
                </div>
                {renderRightElement && renderRightElement(ex)}
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
}

// --- SUB-COMPONENTS ---

function WorkoutsView({ workouts, exercises, units, selectedDate, setSelectedDate, isCalendarExpanded, setIsCalendarExpanded, setEditingWorkout, syncWithFirebase }) {
  const [viewDate, setViewDate] = useState(new Date(selectedDate));
  const currentWorkouts = workouts.filter(w => w.date === selectedDate);
  const workoutDays = workouts.map(w => w.date);

  const days = useMemo(() => {
    if (isCalendarExpanded) return getMonthDays(viewDate.getFullYear(), viewDate.getMonth());
    const curr = new Date(selectedDate);
    const day = curr.getDay();
    curr.setDate(curr.getDate() - day + (day === 0 ? -6 : 1));
    const week = [];
    for (let i = 0; i < 7; i++) { week.push(new Date(curr)); curr.setDate(curr.getDate() + 1); }
    return week;
  }, [isCalendarExpanded, viewDate, selectedDate]);

  return (
    <div className="space-y-6">
      <div className="bg-white dark:bg-gray-800 rounded-3xl p-5 shadow-sm transition-all duration-300 border border-gray-100 dark:border-gray-700/50">
        <div className="flex items-center justify-between mb-6">
          <h2 className="text-lg font-black text-gray-800 dark:text-white uppercase">{viewDate.toLocaleString('default', { month: 'long' })} <span className="text-blue-500">{viewDate.getFullYear()}</span></h2>
          <div className="flex gap-3">
            <div className="flex bg-gray-100 dark:bg-gray-700 p-1 rounded-xl">
              <button onClick={() => { const d = new Date(viewDate); d.setMonth(d.getMonth() - 1); setViewDate(d); }} className="p-1.5 hover:bg-white dark:hover:bg-gray-600 rounded-lg transition-all"><ChevronLeft size={18} /></button>
              <button onClick={() => { const d = new Date(viewDate); d.setMonth(d.getMonth() + 1); setViewDate(d); }} className="p-1.5 hover:bg-white dark:hover:bg-gray-600 rounded-lg transition-all"><ChevronRight size={18} /></button>
            </div>
            <button onClick={() => setIsCalendarExpanded(!isCalendarExpanded)} className={`p-2.5 rounded-xl transition-all ${isCalendarExpanded ? 'bg-blue-500 text-white shadow-sm shadow-blue-200' : 'bg-gray-100 dark:bg-gray-700 text-gray-400'}`}>
              {isCalendarExpanded ? <LayoutGrid size={20} /> : <CalendarDays size={20} />}
            </button>
          </div>
        </div>
        <div className="grid grid-cols-7 gap-2 text-center">
          {['Пн', 'Вт', 'Ср', 'Чт', 'Пт', 'Сб', 'Вс'].map(d => <div key={d} className="text-[11px] text-gray-400 font-black uppercase mb-2">{d}</div>)}
          {days.map((date, i) => {
            const dateStr = formatDate(date);
            const isSelected = dateStr === selectedDate;
            return (
              <button key={i} onClick={() => setSelectedDate(dateStr)} className={`relative h-11 rounded-2xl flex flex-col items-center justify-center transition-all ${isSelected ? 'bg-blue-500 text-white shadow-sm' : 'hover:bg-gray-50 dark:hover:bg-gray-700 text-gray-700 dark:text-gray-300'}`}>
                <span className="text-sm font-bold">{date.getDate()}</span>
                {workoutDays.includes(dateStr) && <div className={`absolute bottom-1.5 w-1 h-1 rounded-full ${isSelected ? 'bg-white' : 'bg-blue-500'}`} />}
              </button>
            );
          })}
        </div>
      </div>
      
      <div className="space-y-6 pb-20">
        <div className="flex items-center justify-between px-1">
          <h3 className="font-black text-gray-800 dark:text-white uppercase tracking-widest text-xs opacity-50">Дневник активности</h3>
          <button onClick={() => setEditingWorkout({ id: Date.now().toString(), date: selectedDate, sets: [] })} className="bg-blue-500 text-white px-5 py-2.5 rounded-2xl text-[10px] font-black flex items-center gap-2 uppercase shadow-sm active:scale-95 transition-all"><Plus size={14} /> Добавить сессию</button>
        </div>
        
        {currentWorkouts.length === 0 ? (
          <div className="bg-gray-50/50 dark:bg-gray-900/30 border-2 border-dashed border-gray-200 dark:border-gray-800 rounded-3xl p-16 text-center text-gray-400 animate-in fade-in">
            <History size={48} className="mx-auto mb-4 opacity-10" />
            <p className="text-xs font-black uppercase tracking-widest">Нет записей за сегодня</p>
          </div>
        ) : (
          currentWorkouts.map(workout => {
            const groupedSets = workout.sets.reduce((acc, set) => {
              const exId = set.exerciseId;
              if (!acc[exId]) acc[exId] = [];
              acc[exId].push(set);
              return acc;
            }, {});

            return (
              <div key={workout.id} className="bg-white dark:bg-gray-800 rounded-[2.5rem] p-6 border border-gray-100 dark:border-gray-700/50 shadow-sm animate-in slide-in-from-bottom-2">
                <div className="flex justify-between items-start mb-6">
                  <span className="text-[10px] font-black uppercase text-gray-400 bg-gray-50 dark:bg-gray-700 px-2.5 py-1 rounded-full">
                    {workout.sets.length} подходов
                  </span>
                  <div className="flex gap-2">
                    <button onClick={() => setEditingWorkout(workout)} className="p-1.5 text-gray-400 hover:text-blue-500 transition-colors"><Edit2 size={14} /></button>
                    <button onClick={() => syncWithFirebase({ workouts: workouts.filter(w => w.id !== workout.id) })} className="p-1.5 text-gray-400 hover:text-red-500 transition-colors"><Trash2 size={14} /></button>
                  </div>
                </div>

                <div className="space-y-6">
                  {Object.entries(groupedSets).map(([exId, sets]) => {
                    const exercise = exercises.find(e => e.id === exId);
                    return (
                      <div key={exId} className="space-y-3">
                        <div className="flex items-baseline gap-2 ml-1">
                          <p className="text-[11px] font-black uppercase tracking-widest text-gray-400 dark:text-gray-500">{exercise?.name || "Упражнение"}</p>
                          {exercise?.equipment && <span className="text-[8px] font-black uppercase px-1.5 py-0.5 rounded bg-blue-500/10 text-blue-500 border border-blue-500/20">{exercise.equipment}</span>}
                        </div>
                        <div className="flex flex-wrap gap-2">
                          {sets.map((set, idx) => (
                            <div key={idx} className={`px-4 py-2 rounded-2xl text-xs font-black transition-all flex items-center gap-1.5 ${set.completed ? 'bg-blue-500/10 text-blue-500 border border-blue-500/20' : 'bg-gray-100 dark:bg-gray-700/50 text-gray-500'}`}>
                              <span className="opacity-50 text-[9px]">{idx + 1}</span>
                              {set.weight}{units} <span className="opacity-50 text-[10px]">×</span> {set.reps}
                            </div>
                          ))}
                        </div>
                      </div>
                    );
                  })}
                </div>
              </div>
            );
          })
        )}
      </div>
    </div>
  );
}

function StatsView({ workouts, exercises, activityData, units, setSelectedExerciseStats, sortMode, setSortMode }) {
  return (
    <div className="space-y-6 pb-24">
      <div className="bg-gradient-to-br from-blue-500 to-blue-600 rounded-3xl p-6 text-white shadow-sm border border-blue-400/20 shadow-blue-100 dark:shadow-none">
        <p className="text-[10px] font-black uppercase opacity-60 mb-2 tracking-widest">Общее количество сессий</p>
        <p className="text-3xl font-black">{workouts.length}</p>
      </div>
      <div className="bg-white dark:bg-gray-800 rounded-3xl p-6 shadow-sm border border-gray-100 dark:border-gray-700/50">
        <h3 className="font-black text-sm uppercase tracking-widest mb-6 dark:text-white">Активность (6 мес)</h3>
        <div className="h-56 w-full">
          <ResponsiveContainer width="100%" height="100%">
            <BarChart data={activityData}>
              <CartesianGrid strokeDasharray="3 3" vertical={false} strokeOpacity={0.1} />
              <XAxis dataKey="name" axisLine={false} tickLine={false} tick={{fontSize: 10, fontWeight: 'bold', fill: '#9ca3af'}} />
              <Tooltip cursor={{fill: 'rgba(59, 130, 246, 0.05)'}} contentStyle={{ borderRadius: '20px', border: 'none', fontWeight: 'bold' }} />
              <Bar dataKey="count" fill="#3b82f6" radius={[6, 6, 6, 6]} barSize={32} />
            </BarChart>
          </ResponsiveContainer>
        </div>
      </div>
      <div className="space-y-4">
        <div className="flex items-center justify-between ml-1">
          <h3 className="font-black text-[10px] uppercase tracking-widest text-gray-400">Упражнения</h3>
          <CustomSelect label="Вид" value={sortMode} onChange={setSortMode} options={[{ label: 'А–Я', value: 'name' }, { label: 'Мышцы', value: 'muscle' }, { label: 'Инвентарь', value: 'equipment' }]} />
        </div>
        
        <GroupedExerciseList 
          exercises={exercises} sortMode={sortMode} onItemClick={(ex) => setSelectedExerciseStats(ex)}
          renderRightElement={() => (<div className="w-10 h-10 rounded-2xl bg-gray-50 dark:bg-gray-700/50 flex items-center justify-center text-gray-300 transition-colors hover:text-blue-500"><TrendingUp size={18} /></div>)}
        />
      </div>
    </div>
  );
}

function SettingsView({ theme, units, exercises, templates, setEditingTemplate, setEditingExercise, syncWithFirebase, sortMode, setSortMode }) {
  return (
    <div className="space-y-10 pb-32">
      <section className="bg-white dark:bg-gray-800 rounded-3xl p-6 shadow-sm space-y-5 border border-gray-100 dark:border-gray-700/50">
        <div className="flex items-center justify-between">
          <span className="text-xs font-black uppercase text-gray-400 tracking-widest">Оформление</span>
          <button onClick={() => syncWithFirebase({ theme: theme === 'dark' ? 'light' : 'dark' })} className="w-12 h-12 flex items-center justify-center bg-gray-50 dark:bg-gray-700 rounded-2xl transition-transform active:scale-95 shadow-sm">{theme === 'dark' ? <Sun size={20} className="text-yellow-400" /> : <Moon size={20} className="text-blue-500" />}</button>
        </div>
        <div className="flex items-center justify-between pt-5 border-t border-gray-50 dark:border-gray-700">
          <span className="text-xs font-black uppercase text-gray-400 tracking-widest">Единицы</span>
          <div className="flex bg-gray-100 dark:bg-gray-900 rounded-2xl p-1 shadow-inner">
            <button onClick={() => syncWithFirebase({ units: 'kg' })} className={`px-5 py-2 rounded-xl text-xs font-black uppercase transition-all ${units === 'kg' ? 'bg-white dark:bg-gray-600 text-blue-500 shadow-sm' : 'text-gray-400'}`}>кг</button>
            <button onClick={() => syncWithFirebase({ units: 'lbs' })} className={`px-5 py-2 rounded-xl text-xs font-black uppercase transition-all ${units === 'lbs' ? 'bg-white dark:bg-gray-600 text-blue-500 shadow-sm' : 'text-gray-400'}`}>lbs</button>
          </div>
        </div>
      </section>
      
      <section className="space-y-4">
        <div className="flex items-center justify-between px-1"><h3 className="font-black text-xs uppercase tracking-widest text-gray-400">Шаблоны</h3><button onClick={() => setEditingTemplate({ id: Date.now().toString(), name: '', exerciseIds: [] })} className="bg-blue-500 text-white p-2.5 rounded-xl hover:scale-105 transition-all shadow-sm shadow-blue-100 active:scale-95"><PlusCircle size={20} /></button></div>
        <div className="bg-white dark:bg-gray-800 rounded-3xl p-6 shadow-sm space-y-3 border border-gray-100 dark:border-gray-700/50">
          {templates.length === 0 ? <p className="text-center text-xs text-gray-400 py-4 font-black uppercase">Шаблонов пока нет</p> : templates.map(t => (
            <div key={t.id} className="flex items-center justify-between p-4 bg-gray-50 dark:bg-gray-900/50 rounded-2xl group transition-all hover:bg-gray-100 dark:hover:bg-gray-700/50 shadow-sm">
              <div><p className="text-sm font-black dark:text-white uppercase tracking-tight">{t.name || "Без названия"}</p><p className="text-[10px] text-gray-400 font-bold uppercase">{t.exerciseIds?.length || 0} упр.</p></div>
              <div className="flex gap-2">
                <button onClick={() => setEditingTemplate(t)} className="p-2 text-gray-400 hover:text-blue-500 transition-colors"><Edit2 size={18} /></button>
                <button onClick={() => syncWithFirebase({ templates: templates.filter(temp => temp.id !== t.id) })} className="p-2 text-gray-400 hover:text-red-500 transition-colors"><Trash2 size={18} /></button>
              </div>
            </div>
          ))}
        </div>
      </section>
      
      <section className="space-y-4">
        <div className="flex items-center justify-between px-1">
          <h3 className="font-black text-xs uppercase tracking-widest text-gray-400">База упражнений</h3>
          <div className="flex items-center gap-3">
            <CustomSelect label="Фильтр" value={sortMode} onChange={setSortMode} options={[{ label: 'А–Я', value: 'name' }, { label: 'Мышцы', value: 'muscle' }, { label: 'Инвентарь', value: 'equipment' }]} />
            <button onClick={() => setEditingExercise({ name: '', category: 'Грудь', equipment: EQUIPMENT_TYPES[0], id: null })} className="bg-blue-500 text-white p-2.5 rounded-xl transition-all shadow-sm shadow-blue-100 active:scale-95"><Plus size={20} /></button>
          </div>
        </div>
        <GroupedExerciseList 
          exercises={exercises} sortMode={sortMode}
          renderRightElement={(ex) => (
            <div className="flex gap-2">
              <button onClick={() => setEditingExercise(ex)} className="p-2 text-gray-300 hover:text-blue-500 transition-colors"><Edit2 size={16} /></button>
              <button onClick={() => syncWithFirebase({ exercises: exercises.filter(e => e.id !== ex.id) })} className="p-2 text-gray-300 hover:text-red-500 transition-colors"><Trash2 size={16} /></button>
            </div>
          )}
        />
      </section>
    </div>
  );
}

// --- MODAL COMPONENTS ---

function WorkoutEditor({ editingWorkout, setEditingWorkout, workouts, templates, exercises, units, syncWithFirebase, setSelectedDate }) {
  if (!editingWorkout) return null;

  const addSet = (exerciseId) => {
    const setsOfThisEx = editingWorkout.sets.filter(s => s.exerciseId === exerciseId);
    const lastSet = setsOfThisEx[setsOfThisEx.length - 1];
    const newSet = { exerciseId, weight: lastSet ? lastSet.weight : 0, reps: lastSet ? lastSet.reps : 0, completed: false };
    setEditingWorkout({ ...editingWorkout, sets: [...editingWorkout.sets, newSet] });
  };

  const applyTemplate = (template) => {
    const exerciseIds = template.exerciseIds || [];
    const newSets = exerciseIds.map(id => ({ exerciseId: id, weight: 0, reps: 0, completed: false }));
    setEditingWorkout({ ...editingWorkout, sets: [...editingWorkout.sets, ...newSets] });
  };

  const handleSave = async () => {
    const nextWorkouts = workouts.find(w => w.id === editingWorkout.id) ? workouts.map(w => w.id === editingWorkout.id ? editingWorkout : w) : [...workouts, editingWorkout];
    await syncWithFirebase({ workouts: nextWorkouts });
    setSelectedDate(editingWorkout.date);
    setEditingWorkout(null);
  };

  return (
    <div className="fixed inset-0 z-50 bg-white dark:bg-gray-900 overflow-y-auto pb-20 animate-in fade-in slide-in-from-bottom-10">
      <div className="max-w-md mx-auto p-6">
        <div className="flex items-center justify-between mb-8"><button onClick={() => setEditingWorkout(null)} className="p-3 bg-gray-50 dark:bg-gray-800 rounded-2xl shadow-sm text-gray-500 active:scale-90 transition-all"><X size={24} /></button><h2 className="text-xl font-black dark:text-white uppercase tracking-tight text-center">Конструктор</h2><button onClick={handleSave} className="bg-blue-500 text-white px-6 py-3 rounded-2xl font-black text-xs uppercase shadow-sm active:scale-95 transition-all">Готово</button></div>
        <div className="mb-6 flex flex-col gap-2"><label className="text-[10px] font-black uppercase text-gray-400 tracking-widest ml-1 text-center">Дата тренировки</label><div className="relative"><input type="date" value={editingWorkout.date} onChange={(e) => setEditingWorkout({ ...editingWorkout, date: e.target.value })} className="w-full bg-gray-50 dark:bg-gray-900 p-4 rounded-2xl border-0 font-black text-sm dark:text-white outline-none focus:ring-2 focus:ring-blue-500 transition-all shadow-inner" /></div></div>
        <div className="mb-6 flex gap-2 overflow-x-auto pb-2 no-scrollbar">
          {templates.map(t => <button key={t.id} onClick={() => applyTemplate(t)} className="px-4 py-3 bg-blue-50 dark:bg-blue-900/20 text-blue-500 rounded-2xl text-[10px] font-black uppercase whitespace-nowrap border border-blue-100 dark:border-blue-900/30 active:scale-95 transition-all shadow-sm">+ {t.name}</button>)}
        </div>
        <div className="space-y-10 pb-10">
          {Array.from(new Set(editingWorkout.sets.map(s => s.exerciseId))).map(exId => {
            const exercise = exercises.find(e => e.id === exId);
            return (
              <div key={exId} className="space-y-4 animate-in fade-in slide-in-from-left-5 duration-300">
                <div className="flex items-baseline gap-2">
                  <h3 className="font-black text-lg dark:text-white uppercase tracking-tight border-l-4 border-blue-500 pl-3">{exercise?.name || "Упражнение"}</h3>
                  {exercise?.equipment && <span className="text-[9px] font-black uppercase text-gray-500 opacity-60">({exercise.equipment})</span>}
                </div>
                <div className="space-y-3">
                  {editingWorkout.sets.filter(s => s.exerciseId === exId).map((set, idx) => {
                    const realIdx = editingWorkout.sets.indexOf(set);
                    return (
                      <div key={realIdx} className={`flex items-center gap-3 p-4 rounded-3xl transition-all shadow-sm border border-transparent ${set.completed ? 'bg-green-50 dark:bg-green-900/20 border-green-200 dark:border-green-900/50' : 'bg-gray-50 dark:bg-gray-800'}`}>
                        <div className="w-8 h-8 flex items-center justify-center font-black text-gray-300 text-xs">{idx + 1}</div>
                        <div className="flex-1 flex gap-2"><input type="number" value={set.weight === 0 ? '' : set.weight} placeholder="Вес" onFocus={e => e.target.select()} onChange={(e) => { const next = [...editingWorkout.sets]; next[realIdx].weight = Number(e.target.value); setEditingWorkout({ ...editingWorkout, sets: next }); }} className="w-16 bg-transparent border-b-2 border-gray-200 dark:border-gray-700 text-center font-black text-lg outline-none dark:text-white focus:border-blue-500 transition-colors" /><span className="text-gray-300 font-bold self-center">×</span><input type="number" value={set.reps === 0 ? '' : set.reps} placeholder="Повт" onFocus={e => e.target.select()} onChange={(e) => { const next = [...editingWorkout.sets]; next[realIdx].reps = Number(e.target.value); setEditingWorkout({ ...editingWorkout, sets: next }); }} className="w-16 bg-transparent border-b-2 border-gray-200 dark:border-gray-700 text-center font-black text-lg outline-none dark:text-white focus:border-blue-500 transition-colors" /></div>
                        <button onClick={() => { const next = [...editingWorkout.sets]; next[realIdx].completed = !next[realIdx].completed; setEditingWorkout({ ...editingWorkout, sets: next }); }} className={`w-12 h-12 flex items-center justify-center rounded-2xl transition-all active:scale-90 shadow-sm ${set.completed ? 'text-green-500 bg-white dark:bg-gray-700 scale-110' : 'text-gray-300 bg-gray-100 dark:bg-gray-700'}`}>{set.completed ? <CheckCircle2 size={24} /> : <Circle size={24} />}</button>
                        <button onClick={() => setEditingWorkout({ ...editingWorkout, sets: editingWorkout.sets.filter((_, i) => i !== realIdx) })} className="p-2 text-gray-300 hover:text-red-500 transition-colors"><Trash2 size={18} /></button>
                      </div>
                    );
                  })}
                </div>
                <button onClick={() => addSet(exId)} className="w-full py-3 rounded-2xl border-2 border-dashed border-gray-200 dark:border-gray-800 text-[10px] font-black text-gray-400 uppercase tracking-widest hover:border-blue-500 hover:text-blue-500 transition-all">+ Добавить подход</button>
              </div>
            );
          })}
          <div className="grid grid-cols-2 gap-3 pt-10 border-t border-gray-100 dark:border-gray-800 pb-10 text-center">
            <h4 className="col-span-2 text-[10px] font-black text-gray-400 uppercase tracking-widest mb-2">Добавить упражнение</h4>
            {exercises.map(ex => <button key={ex.id} onClick={() => addSet(ex.id)} className="p-4 text-[10px] font-black bg-white dark:bg-gray-800 dark:text-white rounded-2xl uppercase shadow-sm text-left border border-gray-100 dark:border-gray-700 hover:bg-blue-500 hover:text-white transition-all active:scale-95">{ex.name}</button>)}
          </div>
        </div>
      </div>
    </div>
  );
}

function TemplateEditorModal({ editingTemplate, setEditingTemplate, templates, exercises, syncWithFirebase, sortMode, setSortMode }) {
  const [draggedIndex, setDraggedIndex] = useState(null);
  if (!editingTemplate) return null;
  const exerciseIds = editingTemplate.exerciseIds || [];
  
  const handleSave = async () => {
    const nextTemplates = templates.find(t => t.id === editingTemplate.id) ? templates.map(t => t.id === editingTemplate.id ? { ...editingTemplate, exerciseIds } : t) : [...templates, { ...editingTemplate, exerciseIds }];
    await syncWithFirebase({ templates: nextTemplates });
    setEditingTemplate(null);
  };

  const onDragStart = (index) => setDraggedIndex(index);
  const onDragOver = (e) => e.preventDefault();
  const onDrop = (index) => {
    if (draggedIndex === null || draggedIndex === index) return;
    const newIds = [...exerciseIds];
    const [removed] = newIds.splice(draggedIndex, 1);
    newIds.splice(index, 0, removed);
    setEditingTemplate({ ...editingTemplate, exerciseIds: newIds });
    setDraggedIndex(null);
  };

  return (
    <div className="fixed inset-0 z-[60] bg-white dark:bg-gray-900 overflow-y-auto pb-20 animate-in fade-in slide-in-from-bottom-10">
      <div className="max-w-md mx-auto p-6">
        <div className="flex items-center justify-between mb-8"><button onClick={() => setEditingTemplate(null)} className="p-3 bg-gray-50 dark:bg-gray-800 rounded-2xl shadow-sm text-gray-500 active:scale-90 transition-all"><X size={24} /></button><h2 className="text-xl font-black dark:text-white uppercase tracking-tight">Редактор шаблона</h2><button onClick={handleSave} className="bg-blue-500 text-white px-6 py-3 rounded-2xl font-black text-xs uppercase shadow-sm active:scale-95 transition-all">Сохранить</button></div>
        <input type="text" value={editingTemplate.name} onChange={(e) => setEditingTemplate({...editingTemplate, name: e.target.value})} placeholder="Название шаблона..." className="w-full bg-gray-50 dark:bg-gray-800 p-4 rounded-2xl border-0 font-bold dark:text-white outline-none focus:ring-2 focus:ring-blue-500 transition-all shadow-inner text-sm mb-8" />
        
        <div className="space-y-4">
          <h3 className="font-black text-xs uppercase tracking-widest text-gray-400 ml-1">Состав шаблона</h3>
          <div className="space-y-2">
            {exerciseIds.length === 0 ? <p className="text-center py-6 text-gray-500 text-[10px] font-bold uppercase border-2 border-dashed border-gray-100 dark:border-gray-800 rounded-2xl">Добавьте упражнения ниже</p> : exerciseIds.map((id, idx) => {
              const ex = exercises.find(e => e.id === id);
              return (
                <div key={`${id}-${idx}`} draggable onDragStart={() => onDragStart(idx)} onDragOver={onDragOver} onDrop={() => onDrop(idx)} className={`flex items-center justify-between p-4 bg-gray-50 dark:bg-gray-800 rounded-2xl border border-gray-100 dark:border-gray-700 shadow-sm transition-all ${draggedIndex === idx ? 'opacity-40 scale-95' : ''}`}>
                  <div className="flex items-center gap-3">
                    <div className="cursor-grab active:cursor-grabbing text-gray-400 p-1"><GripVertical size={16} /></div>
                    <div>
                      <p className="text-sm font-black dark:text-white uppercase tracking-tight">{ex?.name || "Упражнение"}</p>
                      <span className="text-[9px] font-black uppercase text-blue-500">{ex?.category}</span>
                    </div>
                  </div>
                  <button onClick={() => setEditingTemplate({ ...editingTemplate, exerciseIds: exerciseIds.filter((_, i) => i !== idx) })} className="p-2 text-gray-300 hover:text-red-500 transition-colors"><Trash2 size={18} /></button>
                </div>
              );
            })}
          </div>
        </div>

        <div className="pt-10 border-t border-gray-100 dark:border-gray-800 mt-10">
          <div className="flex items-center justify-between mb-5 px-1">
            <h4 className="text-[10px] font-black text-gray-400 uppercase tracking-widest">База упражнений</h4>
            <CustomSelect label="Вид" value={sortMode} onChange={setSortMode} options={[{ label: 'А–Я', value: 'name' }, { label: 'Мышцы', value: 'muscle' }, { label: 'Инвентарь', value: 'equipment' }]} />
          </div>
          <GroupedExerciseList exercises={exercises} sortMode={sortMode} onItemClick={(ex) => setEditingTemplate({ ...editingTemplate, exerciseIds: [...exerciseIds, ex.id] })} />
        </div>
      </div>
    </div>
  );
}

function ExerciseEditorModal({ editingExercise, setEditingExercise, exercises, syncWithFirebase }) {
  const [form, setForm] = useState({ name: '', category: 'Грудь', equipment: 'Штанга' });

  useEffect(() => {
    if (editingExercise) setForm({ name: editingExercise.name, category: editingExercise.category, equipment: editingExercise.equipment || EQUIPMENT_TYPES[0] });
  }, [editingExercise]);

  if (!editingExercise) return null;

  const handleSave = async () => {
    if (!form.name) return;
    const nextEx = editingExercise.id 
      ? exercises.map(e => e.id === editingExercise.id ? { ...e, ...form } : e)
      : [...exercises, { id: Date.now().toString(), ...form }];
    await syncWithFirebase({ exercises: nextEx });
    setEditingExercise(null);
  };

  return (
    <div className="fixed inset-0 z-[70] bg-black/60 backdrop-blur-sm flex items-center justify-center p-6">
      <div className="bg-white dark:bg-gray-800 rounded-[2.5rem] p-8 w-full max-w-sm shadow-2xl animate-in zoom-in-95 duration-200">
        <h3 className="text-lg font-black dark:text-white uppercase tracking-tight mb-6 text-center">
          {editingExercise.id ? 'Редактировать' : 'Новое упражнение'}
        </h3>
        
        <div className="space-y-4">
          <div className="space-y-1.5">
            <label className="text-[10px] font-black uppercase text-gray-400 tracking-widest ml-1">Название</label>
            <input type="text" value={form.name} onChange={(e) => setForm({...form, name: e.target.value})} placeholder="Напр. Тяга в наклоне..." className="w-full bg-gray-100 dark:bg-gray-900 border-0 rounded-2xl p-4 text-sm font-bold dark:text-white outline-none focus:ring-2 focus:ring-blue-500 shadow-inner" autoFocus />
          </div>

          <div className="space-y-1.5">
            <label className="text-[10px] font-black uppercase text-gray-400 tracking-widest ml-1">Группа мышц</label>
            <div className="relative">
              <select value={form.category} onChange={(e) => setForm({...form, category: e.target.value})} className="w-full bg-gray-100 dark:bg-gray-900 rounded-2xl p-4 font-bold dark:text-white outline-none appearance-none text-xs shadow-inner">
                {MUSCLE_GROUPS.map(g => <option key={g} value={g}>{g}</option>)}
              </select>
              <ChevronDown className="absolute right-4 top-1/2 -translate-y-1/2 text-gray-400 pointer-events-none" size={14} />
            </div>
          </div>

          <div className="space-y-1.5 pb-4">
            <label className="text-[10px] font-black uppercase text-gray-400 tracking-widest ml-1">Инвентарь</label>
            <div className="relative">
              <select value={form.equipment} onChange={(e) => setForm({...form, equipment: e.target.value})} className="w-full bg-gray-100 dark:bg-gray-900 rounded-2xl p-4 font-bold dark:text-white outline-none appearance-none text-xs shadow-inner">
                {EQUIPMENT_TYPES.map(e => <option key={e} value={e}>{e}</option>)}
              </select>
              <ChevronDown className="absolute right-4 top-1/2 -translate-y-1/2 text-gray-400 pointer-events-none" size={14} />
            </div>
          </div>
        </div>

        <div className="flex gap-3 mt-4">
          <button onClick={() => setEditingExercise(null)} className="flex-1 py-4 bg-gray-100 dark:bg-gray-700 text-xs font-black uppercase rounded-2xl transition-all active:scale-95 text-gray-500">Отмена</button>
          <button onClick={handleSave} className="flex-1 py-4 bg-blue-500 text-white text-xs font-black uppercase rounded-2xl shadow-lg active:scale-95 transition-all">Сохранить</button>
        </div>
      </div>
    </div>
  );
}

function ExerciseStatsModal({ selectedExerciseStats, setSelectedExerciseStats, workouts, units, exercises }) {
  if (!selectedExerciseStats) return null;
  const ex = selectedExerciseStats;
  const statsData = useMemo(() => {
    const data = [];
    [...workouts].sort((a, b) => new Date(a.date) - new Date(b.date)).forEach(w => {
      const exSets = w.sets.filter(s => s.exerciseId === ex.id && s.completed);
      if (exSets.length > 0) {
        data.push({ 
          date: w.date.split('-').slice(1).reverse().join('.'), 
          maxWeight: Math.max(...exSets.map(s => Number(s.weight))), 
          volume: exSets.reduce((sum, s) => sum + (Number(s.weight) * Number(s.reps)), 0), 
          sets: exSets 
        });
      }
    });
    return data;
  }, [workouts, ex.id]);

  return (
    <div className="fixed inset-0 z-50 bg-white dark:bg-gray-900 overflow-y-auto pb-10 animate-in fade-in zoom-in-95">
      <div className="max-w-md mx-auto p-6">
        <div className="flex items-center justify-between mb-10"><button onClick={() => setSelectedExerciseStats(null)} className="p-3 bg-gray-50 dark:bg-gray-800 rounded-2xl text-gray-500 active:scale-90 transition-all shadow-sm"><X size={24} /></button><div className="text-center"><h2 className="text-xl font-black dark:text-white uppercase leading-none tracking-tight">{ex.name}</h2><p className="text-[10px] text-blue-500 font-black uppercase mt-1 tracking-widest">{ex.category} {ex.equipment ? `• ${ex.equipment}` : ''}</p></div><div className="w-12" /></div>
        <div className="space-y-8">
          {/* График Максимального веса */}
          <div className="bg-white dark:bg-gray-800 rounded-3xl p-6 shadow-sm border border-gray-100 dark:border-gray-700/50 shadow-sm">
            <h3 className="text-[10px] font-black mb-6 dark:text-white uppercase tracking-widest opacity-60 text-center flex items-center justify-center gap-2">
              <TrendingUp size={12} className="text-blue-500" /> Прогресс веса
            </h3>
            <div className="h-44 w-full">
              <ResponsiveContainer width="100%" height="100%">
                <LineChart data={statsData}>
                  <CartesianGrid strokeDasharray="3 3" vertical={false} strokeOpacity={0.1} />
                  <XAxis dataKey="date" tick={{fontSize: 9, fontWeight: 'bold'}} axisLine={false} tickLine={false} />
                  <Line type="monotone" dataKey="maxWeight" stroke="#3b82f6" strokeWidth={4} dot={{r: 4, fill: '#3b82f6'}} activeDot={{r: 6}} />
                </LineChart>
              </ResponsiveContainer>
            </div>
          </div>

          {/* График Роста объема (тоннажа) упражнения */}
          <div className="bg-white dark:bg-gray-800 rounded-3xl p-6 shadow-sm border border-gray-100 dark:border-gray-700/50 shadow-sm">
            <h3 className="text-[10px] font-black mb-6 dark:text-white uppercase tracking-widest opacity-60 text-center flex items-center justify-center gap-2">
              <Activity size={12} className="text-purple-500" /> Общий объем за сессию
            </h3>
            <div className="h-44 w-full">
              <ResponsiveContainer width="100%" height="100%">
                <BarChart data={statsData}>
                  <CartesianGrid strokeDasharray="3 3" vertical={false} strokeOpacity={0.1} />
                  <XAxis dataKey="date" tick={{fontSize: 9, fontWeight: 'bold'}} axisLine={false} tickLine={false} />
                  <Bar dataKey="volume" fill="#8b5cf6" radius={[4, 4, 4, 4]} barSize={20} />
                </BarChart>
              </ResponsiveContainer>
            </div>
          </div>

          <div className="space-y-4 pb-10">
            <h3 className="font-black text-[10px] uppercase tracking-widest text-gray-400 ml-1">История подходов</h3>
            {statsData.length === 0 ? <p className="text-center font-bold text-gray-400 text-sm uppercase py-10">Нет данных</p> : statsData.reverse().map((day, i) => (
              <div key={i} className="bg-white dark:bg-gray-800 rounded-3xl p-5 border border-gray-100 dark:border-gray-700/50 shadow-sm animate-in fade-in slide-in-from-bottom-2">
                <div className="flex justify-between items-center mb-4"><span className="font-black dark:text-white text-sm">{day.date}</span><span className="text-[10px] font-black text-blue-500 uppercase bg-blue-50 dark:bg-blue-900/30 px-3 py-1 rounded-full shadow-sm">Объем: {day.volume} {units}</span></div>
                <div className="flex flex-wrap gap-2">{day.sets.map((s, si) => <div key={si} className="bg-gray-50 dark:bg-gray-700 px-4 py-2 rounded-2xl text-xs font-black dark:text-gray-200 border border-gray-100 dark:border-gray-600/30 shadow-sm">{s.weight}×{s.reps}</div>)}</div>
              </div>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
}