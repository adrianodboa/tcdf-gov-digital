import React, { useState, useEffect, useReducer, useMemo, useCallback, useRef, memo, Suspense, lazy } from "react";
import { db } from './firebaseConfig'; // Importa a instância do Firestore
import { doc, getDoc, setDoc } from 'firebase/firestore'; // Importa funções do Firestore

// ============================================================================
//   Arquivista Ascent — PAINEL DE APROVAÇÃO (Versão Refatorada, Expandida e Final)
//
//   Esta versão centraliza todo o estado em um único reducer, usando Firebase/Firestore
//   para persistência de dados. Inclui todas as funcionalidades discutidas:
//   1. Aba "Desempenho e Notas" (com desempenho geral, por sessão e notas).
//   2. Cálculo de Tempo Total de Estudo (via Pomodoro).
//   3. Painel para adicionar notas de Simulados e Discursivas.
//   4. Sistema de Conquistas (Badges) e Lembretes.
//   5. Exportação e Importação de Dados (Backup/Compartilhamento).
//   6. Contador de Dinheiro (Visão de Vitória) totalmente integrado.
//   7. Planejador Inteligente de Estudos (baseado em cronograma e anamnese).
//   8. Modo Foco.
// ============================================================================


// ---------- 1. TIPOS, CONSTANTES E DADOS INICIAIS ----------

const SubjectKey = {
    Arquivologia: "Arquivologia",
    DireitoAdmin: "Direito Administrativo",
    DireitoConst: "Direito Constitucional",
    Informatica: "Informática e Dados",
    Ingles: "Inglês",
    Raciocinio: "Raciocínio Lógico",
    Portugues: "Português",
    RegimentoInternoCamara: "Regimento Interno da Câmara dos Deputados",
};

const Bank = {
    FGV: "FGV",
    Cebraspe: "Cebraspe",
    FCC: "FCC",
};

const todayStr = () => new Date().toISOString().slice(0, 10);

const addDays = (dateStr, days) => {
    const date = new Date(dateStr + 'T12:00:00Z');
    date.setUTCDate(date.getUTCDate() + days);
    return date.toISOString().slice(0, 10);
};

// ID temporário do documento no Firestore.
// Em um app real, isso seria o ID do usuário logado.
const USER_FIRESTORE_DOC_ID = "dadosDoUsuarioUnico"; 

const INITIAL_STATE = {
    subjects: {
        [SubjectKey.Arquivologia]: { name: SubjectKey.Arquivologia, totalLessons: 17, doneLessons: 0, weight: 2.5 },
        [SubjectKey.Portugues]: { name: SubjectKey.Portugues, totalLessons: 14, doneLessons: 0, weight: 1.5 },
        [SubjectKey.DireitoAdmin]: { name: SubjectKey.DireitoAdmin, totalLessons: 23, doneLessons: 0, weight: 1.5 },
        [SubjectKey.DireitoConst]: { name: SubjectKey.DireitoConst, totalLessons: 26, doneLessons: 0, weight: 1.5 },
        [SubjectKey.Informatica]: { name: SubjectKey.Informatica, totalLessons: 26, doneLessons: 0, weight: 1.0 },
        [SubjectKey.Raciocinio]: { name: SubjectKey.Raciocinio, totalLessons: 10, doneLessons: 0, weight: 1.0 },
        [SubjectKey.Ingles]: { name: SubjectKey.Ingles, totalLessons: 9, doneLessons: 0, weight: 0.8 },
        [SubjectKey.RegimentoInternoCamara]: { name: SubjectKey.RegimentoInternoCamara, totalLessons: 12, doneLessons: 0, weight: 2.0 },
    },
    revisions: {},
    bank: Bank.FGV,
    windows: {
        morning: { start: "05:00", end: "07:00" },
        afternoon: { start: "20:00", end: "22:00" },
    },
    dailyPlan: {}, // Objeto para armazenar planos por data, ex: { "2023-10-26": { morning: [...], afternoon: [...] } }
    checkins: [],
    goalText: "",
    theme: 'system',
    completedDays: 0,
    lastClickDate: "",
    performance: {
        correct: 0,
        wrong: 0,
        studySeconds: 0,
        dailyPerformance: [], // Ex: [{ date: '2025-10-14', correct: 20, wrong: 5 }]
    },
    grades: {
        essays: [],
        mocks: [],
    },
    studyStreak: 0,
    achievements: {
        firstDay: false,
        fiveDayStreak: false,
        tenDayStreak: false,
        arquivologiaMaster: false, // Exemplo: 100% em Arquivologia
    },
    reminders: [],
};

const BANK_TIPS = {
    [Bank.FGV]: {
        title: "FGV — Leitura Crítica e Casos",
        bullets: [
            "Priorize interpretação profunda e aplicações práticas.",
            "Português: textos longos; atenção a semântica e coesão.",
            "Direito: evite decoreba; foque em raciocínio e casos.",
        ],
    },
    [Bank.Cebraspe]: {
        title: "Cebraspe — Itens Certo/Errado",
        bullets: [
            "Um errado pode anular um certo: marque só com segurança.",
            "Cuidado com termos absolutos (sempre, nunca).",
            "Reforce jurisprudência e letra da lei com precisão.",
        ],
    },
    [Bank.FCC]: {
        title: "FCC — Tradicional Evoluída",
        bullets: [
            "Objetividade com interpretação.",
            "Português: pontuação, regência, concordância.",
            "Direito: conceitos, princípios e aplicação textual.",
        ],
    },
};

const WEEKLY_SCHEDULE = {
    0: { morning: null, afternoon: null, notes: "Domingo: Correção do Simulado e Descanso." }, // Domingo
    1: { morning: SubjectKey.Portugues, afternoon: SubjectKey.Arquivologia }, // Segunda
    2: { morning: SubjectKey.DireitoAdmin, afternoon: SubjectKey.DireitoConst }, // Terça
    3: { morning: SubjectKey.Informatica, afternoon: SubjectKey.Portugues }, // Quarta
    4: { morning: SubjectKey.Raciocinio, afternoon: SubjectKey.Arquivologia }, // Quinta
    5: { morning: SubjectKey.Ingles, afternoon: 'Revisão Geral' }, // Sexta
    6: { morning: null, afternoon: null, notes: "Sábado: Simulado ou Bloco de Questões (3-4 horas)." }, // Sábado
};

// ---------- 2. REDUCER CENTRALIZADO & HOOKS ----------

function appReducer(state, action) {
    switch (action.type) {
        case 'SET_STATE':
            // Merge o estado carregado com o estado inicial para garantir que novas chaves existam
            return { ...INITIAL_STATE, ...action.payload };

        case "INCREMENT_LESSON": {
            const s = state.subjects[action.payload];
            if (!s || s.doneLessons >= s.totalLessons) return state;
            const newSubjects = { ...state.subjects, [action.payload]: { ...s, doneLessons: s.doneLessons + 1 } };

            const today = todayStr();
            const d1 = addDays(today, 1);
            const d7 = addDays(today, 7);
            const d30 = addDays(today, 30);
            const newRevs = { ...state.revisions };
            [d1, d7, d30].forEach(date => {
                if (!newRevs[date]) newRevs[date] = [];
                if (!newRevs[date].includes(action.payload)) {
                    newRevs[date].push(action.payload);
                }
            });

            let newAchievements = { ...state.achievements };
            if (action.payload === SubjectKey.Arquivologia && newSubjects[SubjectKey.Arquivologia].doneLessons >= newSubjects[SubjectKey.Arquivologia].totalLessons && !newAchievements.arquivologiaMaster) {
                newAchievements.arquivologiaMaster = true;
            }

            return { ...state, subjects: newSubjects, revisions: newRevs, achievements: newAchievements };
        }
        case "DECREMENT_LESSON": {
            const s = state.subjects[action.payload];
            if (!s || s.doneLessons <= 0) return state;
            const newSubjects = { ...state.subjects, [action.payload]: { ...s, doneLessons: s.doneLessons - 1 } };
            return { ...state, subjects: newSubjects };
        }
        case "SAVE_CHECKIN": {
            const newEntry = { ...action.payload, date: todayStr() };
            const updated = [...state.checkins.filter(c => c.date !== todayStr()), newEntry]
                .sort((a, b) => b.date.localeCompare(a.date));
            return { ...state, checkins: updated.slice(0, 30) };
        }
        case "COMPLETE_DAY": {
            if (state.lastClickDate === todayStr()) return state;

            const yesterday = new Date();
            yesterday.setDate(yesterday.getDate() - 1);
            const yesterdayStr = yesterday.toISOString().slice(0, 10);

            const newStreak = state.lastClickDate === yesterdayStr ? state.studyStreak + 1 : 1;

            let newAchievements = { ...state.achievements };
            if (!newAchievements.firstDay) newAchievements.firstDay = true;
            if (newStreak >= 5 && !newAchievements.fiveDayStreak) newAchievements.fiveDayStreak = true;
            if (newStreak >= 10 && !newAchievements.tenDayStreak) newAchievements.tenDayStreak = true;

            return {
                ...state,
                completedDays: state.completedDays + 1,
                lastClickDate: todayStr(),
                studyStreak: newStreak,
                achievements: newAchievements
            };
        }
        case "SET_DAILY_PLAN": {
            return { ...state, dailyPlan: { ...(state.dailyPlan || {}), [todayStr()]: action.payload } };
        }
        case "TOGGLE_TASK_COMPLETED": {
            const { planDate, windowType, itemIndex, taskIndex } = action.payload;
            const plan = state.dailyPlan[planDate];
            if (!plan || !plan[windowType] || !plan[windowType][itemIndex]) return state;

            // Cria uma cópia profunda para não mutar o estado diretamente
            const updatedDailyPlan = JSON.parse(JSON.stringify(state.dailyPlan));
            const targetItem = updatedDailyPlan[planDate][windowType][itemIndex];

            if (targetItem && targetItem.tasks && targetItem.tasks[taskIndex]) {
                targetItem.tasks[taskIndex].completed = !targetItem.tasks[taskIndex].completed;
            }

            return { ...state, dailyPlan: updatedDailyPlan };
        }
        case "UPDATE_PERFORMANCE": {
            const { key, value } = action.payload; // key: 'correct' ou 'wrong'
            const newTotal = Math.max(0, state.performance[key] + value);

            const today = todayStr();
            let updatedDailyPerformance = [...state.performance.dailyPerformance];
            let currentDailyIndex = updatedDailyPerformance.findIndex(dp => dp.date === today);

            if (currentDailyIndex !== -1) {
                updatedDailyPerformance[currentDailyIndex] = {
                    ...updatedDailyPerformance[currentDailyIndex],
                    [key]: Math.max(0, (updatedDailyPerformance[currentDailyIndex][key] || 0) + value)
                };
            } else {
                updatedDailyPerformance.push({
                    date: today,
                    correct: key === 'correct' ? value : 0,
                    wrong: key === 'wrong' ? value : 0
                });
            }

            return {
                ...state,
                performance: {
                    ...state.performance,
                    [key]: newTotal,
                    dailyPerformance: updatedDailyPerformance
                }
            };
        }
        case "ADD_STUDY_TIME": {
            return { ...state, performance: { ...state.performance, studySeconds: state.performance.studySeconds + action.payload } };
        }
        case "ADD_GRADE": {
            const { type, grade } = action.payload; // type: 'essays' ou 'mocks'
            const newGrade = { ...grade, id: Date.now() };
            const updatedGrades = [...state.grades[type], newGrade].sort((a, b) => b.date.localeCompare(a.date));
            return { ...state, grades: { ...state.grades, [type]: updatedGrades } };
        }
        case "DELETE_GRADE": {
            const { type, id } = action.payload;
            const updatedGrades = state.grades[type].filter(g => g.id !== id);
            return { ...state, grades: { ...state.grades, [type]: updatedGrades } };
        }
        case "ADD_REMINDER": {
            const newReminder = { ...action.payload, id: Date.now(), dismissed: false };
            return { ...state, reminders: [...state.reminders, newReminder] };
        }
        case "DISMISS_REMINDER": {
            return {
                ...state,
                reminders: state.reminders.map(r => r.id === action.payload ? { ...r, dismissed: true } : r)
            };
        }
        case "SET_FIELD": {
            const { field, value } = action.payload;
            return { ...state, [field]: value };
        }
        default:
            return state;
    }
}

function useVictoryTracker(state, dispatch) {
    const { completedDays, lastClickDate } = state;
    const dailyIncrement = 33000 / 100;
    const currentAmount = useMemo(() => completedDays * dailyIncrement, [completedDays, dailyIncrement]);
    const alreadyCompletedToday = useMemo(() => lastClickDate === todayStr(), [lastClickDate]);

    const handleDayComplete = useCallback(() => {
        dispatch({ type: 'COMPLETE_DAY' });
    }, [dispatch]);

    return {
        currentAmount,
        alreadyCompletedToday,
        handleDayComplete,
        completedDays
    };
}

function usePomodoro(dispatch, initialFocusSec = 50 * 60, initialBreakSec = 10 * 60) {
    const [isRunning, setIsRunning] = useState(false);
    const [isBreak, setIsBreak] = useState(false);
    const [secondsLeft, setSecondsLeft] = useState(initialFocusSec);
    const timerRef = useRef(null);

    useEffect(() => {
        if (!isRunning) {
            if (timerRef.current) clearInterval(timerRef.current);
            return;
        }
        timerRef.current = window.setInterval(() => {
            setSecondsLeft((s) => s - 1);
        }, 1000);
        return () => {
            if (timerRef.current) clearInterval(timerRef.current);
        };
    }, [isRunning]);

    useEffect(() => {
        if (secondsLeft > 0) return;

        if (!isBreak) { // Fim do foco
            dispatch({ type: 'ADD_STUDY_TIME', payload: initialFocusSec });
            setIsBreak(true);
            setSecondsLeft(initialBreakSec);
        } else { // Fim da pausa
            setIsBreak(false);
            setSecondsLeft(initialFocusSec);
        }
    }, [secondsLeft, isBreak, initialFocusSec, initialBreakSec, dispatch]);

    const toggle = useCallback(() => setIsRunning(r => !r), []);
    const reset = () => {
        setIsRunning(false);
        setIsBreak(false);
        setSecondsLeft(initialFocusSec);
    };

    const minutes = String(Math.floor(secondsLeft / 60)).padStart(2, "0");
    const seconds = String(secondsLeft % 60).padStart(2, "0");

    return { minutes, seconds, isRunning, isBreak, toggle, reset };
}

// ---------- 3. COMPONENTES DE UI ----------

const Card = memo(({ title, children, className = "" }) => (
    <div className={`bg-white rounded-lg shadow-sm border border-slate-200 p-4 dark:bg-slate-800 dark:border-slate-700 ${className}`}>
        <h2 className="text-base font-bold text-slate-800 mb-3 dark:text-slate-200">{title}</h2>
        {children}
    </div>
));

const SubjectItem = memo(({ subject, dispatch }) => {
    const pct = subject.totalLessons > 0 ? Math.round((subject.doneLessons / subject.totalLessons) * 100) : 0;

    return (
        <li>
            <div className="flex items-center justify-between">
                <span className="text-sm font-medium">{subject.name}</span>
                <span className="text-xs text-slate-500 dark:text-slate-400">{subject.doneLessons}/{subject.totalLessons} ({pct}%)</span>
            </div>
            <div role="progressbar" aria-valuenow={pct} className="mt-1 h-2 rounded-full bg-slate-200 overflow-hidden dark:bg-slate-700">
                <div className="h-full bg-green-600 transition-all duration-500" style={{ width: `${pct}%` }} />
            </div>
            <div className="mt-2 flex items-center gap-2">
                <button aria-label={`Diminuir uma aula de ${subject.name}`} onClick={() => dispatch({ type: 'DECREMENT_LESSON', payload: subject.name })} className="px-2 py-1 text-xs rounded-md border border-slate-300 hover:bg-slate-100 dark:border-slate-600 dark:hover:bg-slate-700">-1</button>
                <button aria-label={`Aumentar uma aula de ${subject.name}`} onClick={() => dispatch({ type: 'INCREMENT_LESSON', payload: subject.name })} className="px-2 py-1 text-xs rounded-md border border-slate-300 hover:bg-slate-100 dark:border-slate-600 dark:hover:bg-slate-700">+1</button>
            </div>
        </li>
    );
});

const ProgressPanel = memo(({ subjects, dispatch }) => (
    <Card title="Progresso por Disciplina">
        <ul className="space-y-4">
            {Object.values(subjects).map((s) => (
                <SubjectItem key={s.name} subject={s} dispatch={dispatch} />
            ))}
        </ul>
    </Card>
));

const PomodoroPanel = memo(({ pomodoro }) => {
    const { minutes, seconds, isRunning, isBreak, toggle, reset } = pomodoro;
    const [quickNote, setQuickNote] = useState('');

    return (
        <Card title="Pomodoro 50/10">
            <div className={`text-5xl font-bold tabular-nums tracking-tight text-center ${isBreak ? 'text-teal-500' : 'text-slate-800 dark:text-slate-200'}`}>
                {minutes}:{seconds}
            </div>
            <div className="flex items-center justify-center gap-2 mt-3">
                <button onClick={toggle} className="px-4 py-2 text-sm rounded-lg border border-slate-300 dark:border-slate-600 hover:bg-slate-100 dark:hover:bg-slate-700 w-24">{isRunning ? 'Pausar' : 'Iniciar'}</button>
                <button onClick={reset} className="px-4 py-2 text-sm rounded-lg border border-slate-300 dark:border-slate-600 hover:bg-slate-100 dark:hover:bg-slate-700">Resetar</button>
            </div>
            <p className="mt-2 text-xs text-center text-slate-500 dark:text-slate-400">{isBreak ? "Hora da pausa! Recarregue as energias." : "Mantenha o foco. Você consegue!"}</p>
            <div className="mt-4">
                <label className="block text-xs font-medium text-slate-500 dark:text-slate-400 mb-1">Notas Rápidas da Sessão</label>
                <textarea
                    value={quickNote}
                    onChange={(e) => setQuickNote(e.target.value)}
                    placeholder="Anotar um insight, dúvida ou ponto para revisar..."
                    className="w-full h-20 border rounded-md px-2 py-1 text-sm bg-transparent border-slate-300 dark:border-slate-600 focus:ring-2 focus:ring-green-500"
                />
            </div>
        </Card>
    );
});

const WellbeingPanel = memo(({ checkins, dispatch }) => {
    const todayEntry = useMemo(() => checkins.find(c => c.date === todayStr()) || {}, [checkins]);

    const [sleepHours, setSleepHours] = useState(8);
    const [pain, setPain] = useState(0);
    const [energyM, setEnergyM] = useState(7);
    const [energyA, setEnergyA] = useState(5);

    useEffect(() => {
        if (todayEntry.date) {
            setSleepHours(todayEntry.sleepHours);
            setPain(todayEntry.pain);
            setEnergyM(todayEntry.energyMorning);
            setEnergyA(todayEntry.energyAfternoon);
        } else {
            // Resetar para valores padrão se não houver entrada para hoje
            setSleepHours(8);
            setPain(0);
            setEnergyM(7);
            setEnergyA(5);
        }
    }, [todayEntry]);

    const handleSave = () => dispatch({ type: 'SAVE_CHECKIN', payload: { sleepHours, pain, energyMorning: energyM, energyAfternoon: energyA } });

    return (
        <Card title="Check-in de Energia">
            <div className="grid grid-cols-2 gap-3 text-sm">
                <div>
                    <label className="block text-xs text-slate-500 dark:text-slate-400">Sono (h)</label>
                    <input type="number" value={sleepHours} onChange={e => setSleepHours(Number(e.target.value))} className="w-full border rounded-md px-2 py-1 mt-1 bg-transparent border-slate-300 dark:border-slate-600" />
                </div>
                <div>
                    <label className="block text-xs text-slate-500 dark:text-slate-400">Dor (0-10)</label>
                    <input type="number" value={pain} onChange={e => setPain(Number(e.target.value))} className="w-full border rounded-md px-2 py-1 mt-1 bg-transparent border-slate-300 dark:border-slate-600" />
                </div>
                <div>
                    <label className="block text-xs text-slate-500 dark:text-slate-400">Energia Manhã</label>
                    <input type="number" value={energyM} onChange={e => setEnergyM(Number(e.target.value))} className="w-full border rounded-md px-2 py-1 mt-1 bg-transparent border-slate-300 dark:border-slate-600" />
                </div>
                <div>
                    <label className="block text-xs text-slate-500 dark:text-slate-400">Energia Tarde</label>
                    <input type="number" value={energyA} onChange={e => setEnergyA(Number(e.target.value))} className="w-full border rounded-md px-2 py-1 mt-1 bg-transparent border-slate-300 dark:border-slate-600" />
                </div>
            </div>
            <button onClick={handleSave} className="w-full mt-3 rounded-lg bg-slate-700 text-white py-2 text-sm font-semibold hover:bg-slate-800 transition-colors dark:bg-slate-600 dark:hover:bg-slate-500">
                {todayEntry.date ? "Atualizar Check-in" : "Salvar Check-in"}
            </button>
        </Card>
    );
});

const VictoryVisionPanel = memo(({ tracker }) => {
    const { currentAmount, alreadyCompletedToday, handleDayComplete } = tracker;
    return (
        <Card title="Visão de Vitória">
            <div className="text-center">
                <p className="text-sm text-slate-600 dark:text-slate-400">Meta Salarial:</p>
                <p className="text-2xl font-bold text-green-700 dark:text-green-500">R$ 33.000,00</p>
                <hr className="my-3 border-slate-200 dark:border-slate-700" />
                <p className="text-sm text-slate-600 dark:text-slate-400">Acumulado na jornada:</p>
                <p className="text-3xl font-bold text-green-700 dark:text-green-500 my-2">
                    {currentAmount.toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' })}
                </p>
                <button onClick={handleDayComplete} disabled={alreadyCompletedToday} className="w-full mt-2 rounded-lg bg-green-700 text-white py-2 text-sm font-semibold hover:bg-green-800 transition-colors disabled:bg-green-700/50 disabled:cursor-not-allowed">
                    {alreadyCompletedToday ? "Dia de hoje concluído!" : "Concluí o dia de estudo!"}
                </button>
                <p className="text-xs text-slate-500 dark:text-slate-400 mt-2">Cada dia de estudo te aproxima R$ 330,00 do seu objetivo.</p>
            </div>
        </Card>
    );
});

// ---------- 4. NOVOS COMPONENTES ----------

const PerformancePanel = memo(({ performance, dispatch }) => {
    const { correct, wrong, studySeconds } = performance;
    const totalQuestions = correct + wrong;
    const accuracy = totalQuestions > 0 ? ((correct / totalQuestions) * 100) : 0;
    const totalHours = (studySeconds / 3600).toFixed(2);

    const handleUpdate = (key, value) => {
        dispatch({ type: 'UPDATE_PERFORMANCE', payload: { key, value } });
    }

    return (
        <Card title="Desempenho Geral">
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                <div className="space-y-3">
                    <div>
                        <p className="text-sm text-slate-500 dark:text-slate-400">Acertos x Erros</p>
                        <p className="text-2xl font-bold">{correct} <span className="text-green-500">vs</span> {wrong}</p>
                    </div>
                    <div>
                        <p className="text-sm text-slate-500 dark:text-slate-400">Taxa de Acerto</p>
                        <p className="text-2xl font-bold">{accuracy.toFixed(1)}%</p>
                    </div>
                    <div>
                        <p className="text-sm text-slate-500 dark:text-slate-400">Tempo de Foco Total</p>
                        <p className="text-2xl font-bold">{totalHours} horas</p>
                    </div>
                </div>
                <div className="space-y-4">
                    <div>
                        <label className="block text-sm font-medium mb-1">Questões Corretas</label>
                        <div className="flex items-center gap-2">
                            <button onClick={() => handleUpdate('correct', 1)} className="px-3 py-1 text-sm rounded-md border border-slate-300 hover:bg-slate-100 dark:border-slate-600 dark:hover:bg-slate-700">+1</button>
                            <button onClick={() => handleUpdate('correct', 10)} className="px-3 py-1 text-sm rounded-md border border-slate-300 hover:bg-slate-100 dark:border-slate-600 dark:hover:bg-slate-700">+10</button>
                            <button onClick={() => handleUpdate('correct', -1)} className="px-3 py-1 text-sm rounded-md border border-slate-300 hover:bg-slate-100 dark:border-slate-600 dark:hover:bg-slate-700">-1</button>
                        </div>
                    </div>
                    <div>
                        <label className="block text-sm font-medium mb-1">Questões Erradas</label>
                        <div className="flex items-center gap-2">
                            <button onClick={() => handleUpdate('wrong', 1)} className="px-3 py-1 text-sm rounded-md border border-slate-300 hover:bg-slate-100 dark:border-slate-600 dark:hover:bg-slate-700">+1</button>
                            <button onClick={() => handleUpdate('wrong', 5)} className="px-3 py-1 text-sm rounded-md border border-slate-300 hover:bg-slate-100 dark:border-slate-600 dark:hover:bg-slate-700">+5</button>
                            <button onClick={() => handleUpdate('wrong', -1)} className="px-3 py-1 text-sm rounded-md border border-slate-300 hover:bg-slate-100 dark:border-slate-600 dark:hover:bg-slate-700">-1</button>
                        </div>
                    </div>
                </div>
            </div>
            <h3 className="text-lg font-semibold mt-6 mb-2">Visualização do Desempenho</h3>
            <div className="relative w-full h-8 flex justify-center items-center rounded-full overflow-hidden bg-red-300 dark:bg-red-800">
                {totalQuestions > 0 ? (
                    <>
                        <div
                            className="absolute h-full bg-green-500"
                            style={{ width: `${accuracy}%`, left: 0 }}
                            title={`Acertos: ${correct}`}
                        />
                        <div className="relative text-white font-bold text-sm z-10">
                            {accuracy.toFixed(1)}%
                        </div>
                    </>
                ) : (
                    <p className="text-slate-500 dark:text-slate-400">Sem dados ainda.</p>
                )}
            </div>
            <p className="text-center text-sm text-slate-500 dark:text-slate-400 mt-2">
                {correct} Acertos / {wrong} Erros
            </p>
        </Card>
    );
});

const GradesPanel = memo(({ grades, dispatch }) => {
    const [type, setType] = useState('mocks');
    const [date, setDate] = useState(todayStr());
    const [topic, setTopic] = useState('');
    const [grade, setGrade] = useState('');

    const handleAddGrade = (e) => {
        e.preventDefault();
        if (!topic || !grade) {
            alert("Por favor, preencha a descrição e a nota.");
            return;
        }
        dispatch({ type: 'ADD_GRADE', payload: { type, grade: { date, topic, grade: parseFloat(grade) } } });
        setTopic('');
        setGrade('');
    };

    const renderTable = (gradeType, title) => (
        <div>
            <h3 className="text-lg font-semibold mb-2">{title}</h3>
            <div className="overflow-x-auto">
                <table className="min-w-full text-sm text-left">
                    <thead className="bg-slate-100 dark:bg-slate-700">
                        <tr>
                            <th className="p-2">Data</th>
                            <th className="p-2">Descrição</th>
                            <th className="p-2">Nota</th>
                            <th className="p-2">Ação</th>
                        </tr>
                    </thead>
                    <tbody className="divide-y divide-slate-200 dark:divide-slate-700">
                        {grades[gradeType].map(g => (
                            <tr key={g.id}>
                                <td className="p-2">{g.date}</td>
                                <td className="p-2">{g.topic}</td>
                                <td className="p-2 font-bold">{g.grade.toFixed(2)}</td>
                                <td className="p-2">
                                    <button onClick={() => dispatch({ type: 'DELETE_GRADE', payload: { type: gradeType, id: g.id } })} className="text-red-500 hover:text-red-700 text-xs">Excluir</button>
                                </td>
                            </tr>
                        ))}
                    </tbody>
                </table>
            </div>
        </div>
    );

    return (
        <Card title="Notas de Simulados e Discursivas">
            <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                <form onSubmit={handleAddGrade} className="md:col-span-1 space-y-4">
                    <h3 className="text-lg font-semibold">Adicionar Nova Nota</h3>
                    <div>
                        <label className="block text-sm font-medium mb-1">Tipo</label>
                        <select value={type} onChange={e => setType(e.target.value)} className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600">
                            <option value="mocks">Simulado</option>
                            <option value="essays">Discursiva</option>
                        </select>
                    </div>
                    <div>
                        <label className="block text-sm font-medium mb-1">Data</label>
                        <input type="date" value={date} onChange={e => setDate(e.target.value)} className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600" />
                    </div>
                    <div>
                        <label className="block text-sm font-medium mb-1">Descrição</label>
                        <input type="text" placeholder="Ex: Simulado 01 - Estratégia" value={topic} onChange={e => setTopic(e.target.value)} className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600" />
                    </div>
                    <div>
                        <label className="block text-sm font-medium mb-1">Nota</label>
                        <input type="number" step="0.1" placeholder="Ex: 8.5" value={grade} onChange={e => setGrade(e.target.value)} className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600" />
                    </div>
                    <button type="submit" className="w-full rounded-lg bg-green-700 text-white py-2 text-sm font-semibold hover:bg-green-800">Adicionar Nota</button>
                </form>

                <div className="md:col-span-2 space-y-6">
                    {renderTable('mocks', 'Simulados')}
                    {renderTable('essays', 'Discursivas')}
                </div>
            </div>
        </Card>
    );
});

const AchievementsPanel = memo(({ achievements }) => (
    <Card title="Suas Conquistas">
        <div className="grid grid-cols-2 gap-4 text-center">
            <div className={`p-3 rounded-lg ${achievements.firstDay ? 'bg-yellow-100 dark:bg-yellow-900/30 text-yellow-800 dark:text-yellow-300' : 'bg-slate-100 dark:bg-slate-700 text-slate-500'}`}>
                <p className="font-bold">Primeiro Dia!</p>
                <p className="text-xs">Iniciou sua jornada.</p>
            </div>
            <div className={`p-3 rounded-lg ${achievements.fiveDayStreak ? 'bg-yellow-100 dark:bg-yellow-900/30 text-yellow-800 dark:text-yellow-300' : 'bg-slate-100 dark:bg-slate-700 text-slate-500'}`}>
                <p className="font-bold">Streak de 5 Dias</p>
                <p className="text-xs">Consistência é chave!</p>
            </div>
            <div className={`p-3 rounded-lg ${achievements.tenDayStreak ? 'bg-yellow-100 dark:bg-yellow-900/30 text-yellow-800 dark:text-yellow-300' : 'bg-slate-100 dark:bg-slate-700 text-slate-500'}`}>
                <p className="font-bold">Streak de 10 Dias</p>
                <p className="text-xs">Imparável!</p>
            </div>
            <div className={`p-3 rounded-lg ${achievements.arquivologiaMaster ? 'bg-yellow-100 dark:bg-yellow-900/30 text-yellow-800 dark:text-yellow-300' : 'bg-slate-100 dark:bg-slate-700 text-slate-500'}`}>
                <p className="font-bold">Mestre em Arquivologia</p>
                <p className="text-xs">Disciplina concluída!</p>
            </div>
        </div>
    </Card>
));

const LogPerformancePanel = memo(({ subjects, dispatch }) => {
    const [subjectKey, setSubjectKey] = useState(Object.keys(subjects)[0]);
    const [correct, setCorrect] = useState('');
    const [wrong, setWrong] = useState('');

    const handleLog = () => {
        if (!correct && !wrong) return;
        dispatch({ type: 'UPDATE_PERFORMANCE', payload: { key: 'correct', value: Number(correct) } });
        dispatch({ type: 'UPDATE_PERFORMANCE', payload: { key: 'wrong', value: Number(wrong) } });

        alert(`Desempenho em ${subjects[subjectKey].name} registrado!`);
        setCorrect('');
        setWrong('');
    };

    return (
        <Card title="Registrar Desempenho da Sessão">
            <div className="space-y-3">
                <div>
                    <label className="block text-sm font-medium mb-1">Matéria Estudada</label>
                    <select value={subjectKey} onChange={e => setSubjectKey(e.target.value)} className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600">
                        {Object.values(subjects).map(s => <option key={s.name} value={s.name}>{s.name}</option>)}
                    </select>
                </div>
                <div className="grid grid-cols-2 gap-3">
                    <div>
                        <label className="block text-sm font-medium mb-1">Acertos</label>
                        <input type="number" value={correct} onChange={e => setCorrect(e.target.value)} className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600" />
                    </div>
                    <div>
                        <label className="block text-sm font-medium mb-1">Erros</label>
                        <input type="number" value={wrong} onChange={e => setWrong(e.target.value)} className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600" />
                    </div>
                </div>
                <button onClick={handleLog} className="w-full rounded-lg bg-teal-700 text-white py-2 text-sm font-semibold hover:bg-teal-800">Registrar</button>
            </div>
        </Card>
    );
});

const ReminderSystem = memo(({ reminders, dispatch }) => {
    const [visibleReminders, setVisibleReminders] = useState([]);

    useEffect(() => {
        const interval = setInterval(() => {
            const now = new Date();
            const nowTime = `${String(now.getHours()).padStart(2, '0')}:${String(now.getMinutes()).padStart(2, '0')}`;
            const today = todayStr();

            const dueReminders = reminders.filter(r =>
                !r.dismissed &&
                r.date === today &&
                r.time === nowTime &&
                !visibleReminders.some(vr => vr.id === r.id)
            );

            if (dueReminders.length > 0) {
                setVisibleReminders(prev => [...prev, ...dueReminders]);
            }
        }, 60 * 1000); // Checa a cada minuto

        return () => clearInterval(interval);
    }, [reminders, visibleReminders]);

    const handleDismiss = (id) => {
        dispatch({ type: 'DISMISS_REMINDER', payload: id });
        setVisibleReminders(prev => prev.filter(r => r.id !== id));
    };

    if (visibleReminders.length === 0) return null;

    return (
        <div className="fixed top-20 right-4 z-50 space-y-3">
            {visibleReminders.map(r => (
                <div key={r.id} className="bg-blue-500 text-white p-4 rounded-lg shadow-lg flex items-center justify-between animate-fade-in-down">
                    <div>
                        <p className="font-bold">Lembrete: {r.message}</p>
                        <p className="text-xs">{r.time}</p>
                    </div>
                    <button onClick={() => handleDismiss(r.id)} className="ml-4 text-white hover:text-blue-200">
                        ✕
                    </button>
                </div>
            ))}
        </div>
    );
});

const ReminderAddPanel = memo(({ dispatch }) => {
    const [date, setDate] = useState(todayStr());
    const [time, setTime] = useState('09:00');
    const [message, setMessage] = useState('');

    const handleAddReminder = (e) => {
        e.preventDefault();
        if (!message) {
            alert("A mensagem do lembrete não pode ser vazia.");
            return;
        }
        dispatch({ type: 'ADD_REMINDER', payload: { date, time, message } });
        setMessage('');
        alert("Lembrete adicionado!");
    };

    return (
        <Card title="Adicionar Lembrete">
            <form onSubmit={handleAddReminder} className="space-y-3">
                <div>
                    <label className="block text-sm font-medium mb-1">Data</label>
                    <input type="date" value={date} onChange={e => setDate(e.target.value)} className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600" />
                </div>
                <div>
                    <label className="block text-sm font-medium mb-1">Hora</label>
                    <input type="time" value={time} onChange={e => setTime(e.target.value)} className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600" />
                </div>
                <div>
                    <label className="block text-sm font-medium mb-1">Mensagem</label>
                    <input type="text" value={message} onChange={e => setMessage(e.target.value)} placeholder="Ex: Revisar súmulas STF" className="w-full border rounded-md px-2 py-1 bg-transparent border-slate-300 dark:border-slate-600" />
                </div>
                <button type="submit" className="w-full rounded-lg bg-indigo-700 text-white py-2 text-sm font-semibold hover:bg-indigo-800">Adicionar Lembrete</button>
            </form>
        </Card>
    );
});

const LazySummaryContent = lazy(async () => {
    const SummaryContentComponent = memo(() => (
        <div className="space-y-6 animate-fade-in">
            <Card title="Resumo do Plano de Aprovação (Alana)">
                <div className="prose prose-sm max-w-none prose-slate dark:prose-invert">
                    <p>Esta é uma visão geral da sua estratégia de estudos, focada nos seus pontos fortes e desafios para garantir uma preparação de alto nível.</p>

                    <h3>Técnicas de Estudo e Princípios Gerais</h3>
                    <ul>
                        <li><strong>Metodologia (PDF/Resumo + Questões):</strong> A base do estudo será a leitura de PDFs e resumos como fonte principal, combinada com a prática massiva de questões (estudo reverso).</li>
                        <li><strong>Rotina e Energia:</strong> O plano considera 20h de estudo durante a semana e 8h no fim de semana, com foco nas janelas de 05-07h e 20-22h. A técnica Pomodoro (50/10) é usada para gerenciar o foco.</li>
                        <li><strong>Revisão (SRS):</strong> O sistema de revisão espaçada (SRS 1-7-30) é integrado e automatizado para garantir a retenção do conteúdo a longo prazo.</li>
                        <li><strong>Estudo Reverso:</strong> Após a primeira passagem pelo conteúdo, as sessões de revisão devem começar diretamente pelas questões no tecconcursos. Erros e dúvidas guiam o estudo no material teórico.</li>
                    </ul>
                    <h3>Simulados e Provas Discursivas</h3>
                    <p>A prática simulando o dia da prova é um pilar fundamental do plano.</p>
                    <ul>
                        <li><strong>Simulado (Sábado):</strong> A cada 15 dias, um simulado completo (tempo de prova, sem consultas). Nos sábados sem simulado, blocos de questões de 3 horas.</li>
                        <li><strong>Correção (Domingo):</strong> Momento mais importante da semana. Análise detalhada de cada erro para entender a causa raiz e revisar o tópico correspondente.</li>
                        <li><strong>Prova Discursiva (FGV):</strong> A banca costuma apresentar um problema prático relacionado ao cargo. A resposta deve ser técnica, clara, objetiva e bem fundamentada. A prática regular é recomendada.</li>
                    </ul>
                </div>
            </Card>
            <Card title="Cronograma Semanal Sugerido (28h semanais)">
                <div className="overflow-x-auto">
                    <table className="min-w-full text-sm text-left table-auto">
                        <thead className="bg-slate-200 dark:bg-slate-700">
                            <tr>
                                <th className="p-2">Dia</th><th className="p-2">05h-07h</th><th className="p-2">Livre</th><th className="p-2">20h-22h</th><th className="p-2">Livre</th>
                            </tr>
                        </thead>
                        <tbody className="divide-y divide-slate-200 dark:divide-slate-700">
                            <tr><td className="p-2 font-bold">Segunda</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Português</td><td className="p-2">Livre</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Arquivologia</td><td className="p-2">Livre</td></tr>
                            <tr><td className="p-2 font-bold">Terça</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Dir. Admin.</td><td className="p-2">Livre</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Dir. Const.</td><td className="p-2">Livre</td></tr>
                            <tr><td className="p-2 font-bold">Quarta</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Informática</td><td className="p-2">Livre</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Português</td><td className="p-2">Livre</td></tr>
                            <tr><td className="p-2 font-bold">Quinta</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Raciocínio L.</td><td className="p-2">Livre</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Arquivologia</td><td className="p-2">Livre</td></tr>
                            <tr><td className="p-2 font-bold">Sexta</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Inglês</td><td className="p-2">Livre</td><td className="p-2 bg-green-50 dark:bg-green-900/30">Revisão Geral</td><td className="p-2">Livre</td></tr>
                            <tr className="bg-blue-50 dark:bg-blue-900/30"><td className="p-2 font-bold">Sábado</td><td className="p-2" colSpan={4}>Simulado ou Bloco de Questões (3-4 horas)</td></tr>
                            <tr className="bg-blue-50 dark:bg-blue-900/30"><td className="p-2 font-bold">Domingo</td><td className="p-2" colSpan={4}>Correção do Simulado e Descanso</td></tr>
                        </tbody>
                    </table>
                </div>
            </Card>
        </div>
    ));
    return { default: SummaryContentComponent };
});
const SummaryContent = (props) => <Suspense fallback={<div className="p-4">Carregando resumo...</div>}><LazySummaryContent {...props} /></Suspense>;


// ---------- 5. COMPONENTE PRINCIPAL (APP) ----------

export default function ArquivistaAscentApp() {
    const [activeTab, setActiveTab] = useState('dashboard');
    const [isFocusMode, setFocusMode] = useState(false);

    const [state, dispatch] = useReducer(appReducer, INITIAL_STATE);
    const { subjects, revisions, bank, windows, dailyPlan, checkins, goalText, theme, completedDays, lastClickDate, performance, grades, achievements, reminders } = state;

    // --- Firebase: Carregar Estado ---
    useEffect(() => {
        const loadData = async () => {
            const docRef = doc(db, "usuarios", USER_FIRESTORE_DOC_ID);
            const docSnap = await getDoc(docRef);

            if (docSnap.exists()) {
                console.log("Dados carregados do Firestore!");
                const loadedState = docSnap.data();
                dispatch({ type: 'SET_STATE', payload: loadedState });
            } else {
                console.log("Nenhum documento encontrado no Firestore, usando estado inicial.");
            }
        };
        loadData();
    }, []); // Executa apenas uma vez ao montar

    // --- Firebase: Salvar Estado ---
    useEffect(() => {
        const saveData = async () => {
            // Verifica se o estado é diferente do inicial para evitar gravação desnecessária na inicialização
            if (JSON.stringify(state) !== JSON.stringify(INITIAL_STATE)) {
                try {
                    await setDoc(doc(db, "usuarios", USER_FIRESTORE_DOC_ID), state);
                    console.log("Dados salvos no Firestore!");
                } catch (e) {
                    console.error("Erro ao salvar dados no Firestore: ", e);
                }
            }
        };
        // Pequeno delay para evitar salvar a cada pequena mudança, mas garantir persistência
        const handler = setTimeout(() => {
            saveData();
        }, 500); // Salva 500ms após a última mudança de estado

        return () => clearTimeout(handler); // Limpa o timer se o estado mudar novamente
    }, [state]); // Salva sempre que o estado completo muda

    // --- Efeito para o Tema (Dark Mode / Tema Câmara) ---
    useEffect(() => {
        const root = document.documentElement;
        root.classList.remove('dark', 'theme-camara');

        if (theme === 'dark' || (theme === 'system' && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
            root.classList.add('dark');
        }
        if (theme === 'camara') {
            root.classList.add('theme-camara');
        }
    }, [theme]);

    // --- Hooks e Memos Derivados ---
    const pomodoro = usePomodoro(dispatch);
    const victoryTracker = useVictoryTracker(state, dispatch);

    const todayRevisions = useMemo(() => revisions[todayStr()] || [], [revisions]);
    const planForToday = useMemo(() => (dailyPlan || {})[todayStr()], [dailyPlan]);

    // --- Lógica de Geração do Plano Inteligente ---
    const generatePlan = useCallback(() => {
        if (planForToday) {
            alert("Já existe um plano gerado para hoje. Conclua-o ou reinicie a página se desejar gerar um novo (dados serão salvos).");
            return;
        }

        const today = new Date();
        const dayOfWeek = today.getDay(); // 0 = Domingo, 1 = Segunda...
        const scheduleForToday = WEEKLY_SCHEDULE[dayOfWeek];

        if (!scheduleForToday) {
            dispatch({ type: 'SET_DAILY_PLAN', payload: { morning: [], afternoon: [], notes: "Dia sem plano definido." } });
            return;
        }

        if (scheduleForToday.notes) {
            dispatch({ type: 'SET_DAILY_PLAN', payload: { morning: [], afternoon: [], notes: scheduleForToday.notes } });
            return;
        }

        const createTasksForSubject = (subjectKey) => {
            if (!subjectKey) return [];

            let tasks = [];
            switch (subjectKey) {
                case SubjectKey.Arquivologia:
                    tasks = [
                        { text: "Estudar 1 tópico base (usar vídeo se o PDF estiver difícil)", completed: false },
                        { text: "Fazer 15 questões sobre o tópico para quebrar a resistência", completed: false },
                        { text: "Criar um mapa mental simples do tópico para visualizar", completed: false }
                    ];
                    break;
                case SubjectKey.Portugues:
                    tasks = [
                        { text: "Revisar 1 resumo existente em 20 min", completed: false },
                        { text: "Resolver 30 questões de bancas variadas (modo turbo)", completed: false },
                        { text: "Anotar 2-3 regras ou exceções que geraram dúvida", completed: false }
                    ];
                    break;
                case SubjectKey.DireitoAdmin:
                    tasks = [
                        { text: "Estudar tópico da Nova Lei de Licitações (Lei 14.133)", completed: false },
                        { text: "Focar em entender a aplicação prática, não apenas decorar", completed: false },
                        { text: "Resolver 10 questões específicas sobre a nova lei", completed: false }
                    ];
                    break;
                case 'Revisão Geral':
                    if (todayRevisions.length > 0) {
                        tasks = todayRevisions.map(revSubject => ({ text: `Revisar ${revSubject} (SRS)`, completed: false }));
                    } else {
                        tasks = [{ text: "Revisão livre: Escolha uma matéria que sentiu mais dificuldade na semana.", completed: false }];
                    }
                    break;
                default:
                    tasks = [
                        { text: `Ler 1 PDF de ${subjectKey}`, completed: false },
                        { text: `Resolver 20 questões (${bank})`, completed: false },
                        { text: "Fichar 3 pontos-chave para o SRS", completed: false }
                    ];
            }
            return tasks;
        };

        const subjectMorning = scheduleForToday.morning;
        const subjectAfternoon = scheduleForToday.afternoon;

        // Função para verificar se a matéria já foi concluída
        const isSubjectDone = (subjectKey) => {
            if (!subjectKey || subjectKey === 'Revisão Geral') return false;
            const subject = subjects[subjectKey];
            return subject && subject.doneLessons >= subject.totalLessons;
        };

        const morningPlan = {
            subject: subjectMorning,
            type: isSubjectDone(subjectMorning) ? "Matéria Concluída (Revisar)" : "Conteúdo Novo",
            tasks: createTasksForSubject(subjectMorning)
        };

        const afternoonPlan = {
            subject: subjectAfternoon,
            type: isSubjectDone(subjectAfternoon) ? "Matéria Concluída (Revisar)" : "Conteúdo Novo",
            tasks: createTasksForSubject(subjectAfternoon)
        };

        dispatch({ type: 'SET_DAILY_PLAN', payload: { morning: [morningPlan], afternoon: [afternoonPlan], notes: "" } });

    }, [subjects, bank, planForToday, dispatch, todayRevisions]);

    const setField = (field, value) => dispatch({ type: 'SET_FIELD', payload: { field, value } });

    // --- Funcionalidades de Exportar/Importar ---
    const handleExportData = () => {
        const stateToExport = JSON.stringify(state, null, 2);
        const blob = new Blob([stateToExport], { type: 'application/json' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `arquivista_ascent_backup_${todayStr()}.json`;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
        alert('Dados exportados com sucesso!');
    };

    const handleImportData = (e) => {
        const file = e.target.files[0];
        if (file) {
            const reader = new FileReader();
            reader.onload = (event) => {
                try {
                    const importedState = JSON.parse(event.target.result);
                    dispatch({ type: 'SET_STATE', payload: importedState });
                    alert('Dados importados com sucesso! A página será recarregada.');
                    window.location.reload();
                } catch (error) {
                    alert('Erro ao importar os dados. Verifique se o arquivo JSON é válido.');
                    console.error('Erro de importação:', error);
                }
            };
            reader.readAsText(file);
        }
    };


    return (
        <div className="min-h-screen bg-slate-100 text-slate-800 font-sans dark:bg-slate-900 dark:text-slate-200">
            {/* --- CSS Embutido para Tema 'Câmara' --- */}
            <style>{`
                .theme-camara {
                    --theme-primary: #008247; /* Verde da bandeira */
                    --theme-secondary: #fec310; /* Amarelo/Dourado */
                    --theme-text-primary: #333;
                    --theme-text-on-primary: #FFFFFF;
                    --theme-bg-primary: #f0f2f5;
                    --theme-bg-secondary: #FFFFFF;
                    --theme-border: #d1d5db;
                }
                .dark .theme-camara {
                    --theme-primary: #00a65a;
                    --theme-secondary: #ffde59;
                    --theme-text-primary: #e5e7eb;
                    --theme-text-on-primary: #111827;
                    --theme-bg-primary: #1a2433;
                    --theme-bg-secondary: #2c3a50;
                    --theme-border: #4b5563;
                }
                .theme-camara .text-green-700 { color: var(--theme-primary); }
                .dark .theme-camara .dark\\:text-green-500 { color: var(--theme-primary); }
                .theme-camara .border-green-700 { border-color: var(--theme-primary); }
                .theme-camara .bg-green-700 { background-color: var(--theme-primary); color: var(--theme-text-on-primary); }
            `}</style>

            <header className="sticky top-0 z-20 backdrop-blur-lg bg-white/80 dark:bg-slate-900/80 border-b border-slate-300 dark:border-slate-700">
                <div className="mx-auto max-w-screen-xl px-4">
                    <div className="flex items-center justify-between py-3">
                        <h1 className="text-xl font-bold tracking-tight text-green-800 dark:text-green-500">Arquivista da Câmara — Alana</h1>
                        <div className="flex items-center gap-4">
                            <button
                                onClick={() => setFocusMode(!isFocusMode)}
                                className="px-3 py-1 text-sm font-medium rounded-full border border-purple-500 text-purple-700 bg-purple-50 hover:bg-purple-100 dark:bg-purple-900/30 dark:text-purple-300"
                            >
                                {isFocusMode ? 'Sair do Foco' : 'Modo Foco'}
                            </button>
                            <span className="text-sm font-medium flex items-center gap-2 border border-orange-400 bg-orange-50 text-orange-700 dark:bg-orange-900/30 dark:border-orange-700 dark:text-orange-300 rounded-full px-3 py-1">
                                🔥 <span className="font-bold">{state.studyStreak}</span> {state.studyStreak === 1 ? 'dia de streak' : 'dias de streak'}
                            </span>
                            <span className="text-sm font-medium border border-amber-400 bg-amber-50 text-amber-700 dark:bg-amber-900/30 dark:border-amber-700 dark:text-amber-300 rounded-full px-3 py-1">
                                Revisões Hoje: {todayRevisions.length}
                            </span>
                            <div className="flex items-center gap-2">
                                <label className="text-sm font-medium">Banca:</label>
                                <select value={bank} onChange={(e) => setField('bank', e.target.value)} className="rounded-md border-slate-300 text-sm px-2 py-1 focus:ring-2 focus:ring-green-600 bg-transparent dark:border-slate-600">
                                    {Object.values(Bank).map(b => <option key={b} value={b}>{b}</option>)}
                                </select>
                            </div>
                        </div>
                    </div>
                    <div role="tablist" aria-label="Navegação principal" className="flex border-b border-slate-300 -mx-4 px-4 dark:border-slate-700">
                        <button role="tab" aria-selected={activeTab === 'dashboard'} onClick={() => setActiveTab('dashboard')} className={`px-4 py-2 text-sm font-medium transition-colors ${activeTab === 'dashboard' ? 'border-b-2 border-green-700 text-green-700 dark:text-green-500' : 'text-slate-500 hover:text-slate-700 dark:hover:text-slate-300'}`}>Painel</button>
                        <button role="tab" aria-selected={activeTab === 'performance'} onClick={() => setActiveTab('performance')} className={`px-4 py-2 text-sm font-medium transition-colors ${activeTab === 'performance' ? 'border-b-2 border-green-700 text-green-700 dark:text-green-500' : 'text-slate-500 hover:text-slate-700 dark:hover:text-slate-300'}`}>Desempenho e Notas</button>
                        <button role="tab" aria-selected={activeTab === 'summary'} onClick={() => setActiveTab('summary')} className={`px-4 py-2 text-sm font-medium transition-colors ${activeTab === 'summary' ? 'border-b-2 border-green-700 text-green-700 dark:text-green-500' : 'text-slate-500 hover:text-slate-700 dark:hover:text-slate-300'}`}>Resumo do Plano</button>
                    </div>
                </div>
            </header>

            <ReminderSystem reminders={reminders} dispatch={dispatch} /> {/* Sistema de Lembretes */}

            <main className="mx-auto max-w-screen-xl p-4">
                {activeTab === 'dashboard' ? (
                    <div className="grid grid-cols-1 lg:grid-cols-12 gap-6 animate-fade-in">
                        <aside className={`lg:col-span-4 xl:col-span-3 space-y-6 ${isFocusMode ? 'hidden lg:block' : ''}`}>
                            <VictoryVisionPanel tracker={victoryTracker} />
                            <ProgressPanel subjects={subjects} dispatch={dispatch} />
                            <Card title="Configurações">
                                <button onClick={generatePlan} disabled={!!planForToday} className="w-full mb-4 rounded-lg bg-blue-900 text-white py-2 text-sm font-semibold hover:bg-blue-800 transition-colors disabled:bg-blue-900/50 disabled:cursor-not-allowed dark:bg-blue-700 dark:hover:bg-blue-600">
                                    {planForToday ? "Plano de Hoje Gerado" : "Gerar Plano Estratégico"}
                                </button>
                                <hr className="my-4 border-slate-200 dark:border-slate-700" />
                                <h3 className="text-sm font-bold mb-2">Gerenciamento de Dados</h3>
                                <div className="flex flex-col gap-2">
                                    <button
                                        onClick={handleExportData}
                                        className="w-full rounded-lg bg-indigo-700 text-white py-2 text-sm font-semibold hover:bg-indigo-800 transition-colors dark:bg-indigo-600 dark:hover:bg-indigo-500"
                                    >
                                        Exportar Dados
                                    </button>
                                    <label htmlFor="import-file" className="w-full cursor-pointer rounded-lg bg-orange-700 text-white py-2 text-sm font-semibold text-center hover:bg-orange-800 transition-colors dark:bg-orange-600 dark:hover:bg-orange-500">
                                        Importar Dados
                                        <input
                                            id="import-file"
                                            type="file"
                                            accept=".json"
                                            className="hidden"
                                            onChange={handleImportData}
                                        />
                                    </label>
                                </div>
                            </Card>
                            <ReminderAddPanel dispatch={dispatch} /> {/* Painel para Adicionar Lembretes */}
                        </aside>

                        <section className={`lg:col-span-8 xl:col-span-9 space-y-6 ${isFocusMode ? 'lg:col-span-12' : ''}`}>
                            <Card title="Planejamento e Metas do Dia" className={`${isFocusMode ? 'hidden' : ''}`}>
                                <input value={goalText} onChange={(e) => setField('goalText', e.target.value)} placeholder="Ex: Ler 1 PDF de Arquivologia + 30 questões" className="w-full border rounded-md px-3 py-2 text-sm bg-transparent border-slate-300 dark:border-slate-600" />
                            </Card>

                            {planForToday && (
                                <Card title="Plano Estratégico Gerado" className={isFocusMode ? 'hidden' : ''}>
                                    <div className="grid grid-cols-1 md:grid-cols-2 gap-4 text-sm">
                                        {planForToday.morning && planForToday.morning.length > 0 && (
                                            <div>
                                                <h3 className="font-bold text-slate-600 dark:text-slate-300 mb-2">Manhã</h3>
                                                <ul className="space-y-3">
                                                    {planForToday.morning.map((p, i) => (
                                                        <li key={i} className="bg-slate-50 border rounded-lg p-3 dark:bg-slate-900/50 dark:border-slate-700">
                                                            <h4 className="font-bold">{p.subject} ({p.type})</h4>
                                                            <ul className="mt-1 text-slate-700 dark:text-slate-300 text-xs space-y-2">
                                                                {(p.tasks || []).map((t, ti) => (
                                                                    <li key={ti} className="flex items-center">
                                                                        <input type="checkbox" id={`task-m-${i}-${ti}`}
                                                                            checked={!!t.completed}
                                                                            onChange={() => dispatch({ type: 'TOGGLE_TASK_COMPLETED', payload: { planDate: todayStr(), windowType: 'morning', itemIndex: i, taskIndex: ti } })}
                                                                            className="h-4 w-4 rounded border-gray-300 text-green-600 focus:ring-green-500" />
                                                                        <label htmlFor={`task-m-${i}-${ti}`} className={`ml-2 ${t.completed ? 'line-through text-slate-400' : ''}`}>
                                                                            {t.text}
                                                                        </label>
                                                                    </li>
                                                                ))}
                                                            </ul>
                                                        </li>
                                                    ))}
                                                </ul>
                                            </div>
                                        )}
                                        {planForToday.afternoon && planForToday.afternoon.length > 0 && (
                                            <div>
                                                <h3 className="font-bold text-slate-600 dark:text-slate-300 mb-2">Noite</h3>
                                                <ul className="space-y-3">
                                                    {planForToday.afternoon.map((p, i) => (
                                                        <li key={i} className="bg-slate-50 border rounded-lg p-3 dark:bg-slate-900/50 dark:border-slate-700">
                                                            <h4 className="font-bold">{p.subject} ({p.type})</h4>
                                                            <ul className="mt-1 text-slate-700 dark:text-slate-300 text-xs space-y-2">
                                                                {(p.tasks || []).map((t, ti) => (
                                                                    <li key={ti} className="flex items-center">
                                                                        <input type="checkbox" id={`task-a-${i}-${ti}`}
                                                                            checked={!!t.completed}
                                                                            onChange={() => dispatch({ type: 'TOGGLE_TASK_COMPLETED', payload: { planDate: todayStr(), windowType: 'afternoon', itemIndex: i, taskIndex: ti } })}
                                                                            className="h-4 w-4 rounded border-gray-300 text-green-600 focus:ring-green-500" />
                                                                        <label htmlFor={`task-a-${i}-${ti}`} className={`ml-2 ${t.completed ? 'line-through text-slate-400' : ''}`}>
                                                                            {t.text}
                                                                        </label>
                                                                    </li>
                                                                ))}
                                                            </ul>
                                                        </li>
                                                    ))}
                                                </ul>
                                            </div>
                                        )}
                                        {planForToday.notes && (
                                            <div className="md:col-span-2 text-center text-slate-500 dark:text-slate-400 p-4">
                                                <p className="font-bold">{planForToday.notes}</p>
                                            </div>
                                        )}
                                    </div>
                                </Card>
                            )}
                            <div className={`grid grid-cols-1 ${isFocusMode ? '' : 'md:grid-cols-2'} gap-6`}>
                                <PomodoroPanel pomodoro={pomodoro} />
                                <div className={isFocusMode ? 'hidden' : ''}>
                                    <WellbeingPanel checkins={checkins} dispatch={dispatch} />
                                </div>
                            </div>
                        </section>
                    </div>
                ) : activeTab === 'performance' ? (
                    <div className="space-y-6 animate-fade-in">
                        <PerformancePanel performance={performance} dispatch={dispatch} />
                        <LogPerformancePanel subjects={subjects} dispatch={dispatch} />
                        <AchievementsPanel achievements={achievements} />
                        <GradesPanel grades={grades} dispatch={dispatch} />
                    </div>
                ) : (
                    <SummaryContent />
                )}
            </main>
        </div>
    );
}
