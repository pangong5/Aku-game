import React, { useState, useEffect, useRef } from 'react';
import {
  Gamepad2,
  Trophy,
  Rocket,
  VolumeX,
  Target,
  Zap,
  Play,
  RotateCcw,
  Sparkles,
  Copy,
  Check,
  Code,
  Flame,
  Award,
  Mic,
  AlertCircle,
} from 'lucide-react';

interface NoiseGameViewProps {
  currentDb: number;
  isMonitoring: boolean;
  onToggleMonitoring: () => void;
  isDemoMode: boolean;
}

type GameMode = 'rocket' | 'quiet' | 'target' | 'scream';

interface HighScores {
  rocket: number;
  quiet: number;
  target: number;
  scream: number;
}

export const NoiseGameView: React.FC<NoiseGameViewProps> = ({
  currentDb,
  isMonitoring,
  onToggleMonitoring,
  isDemoMode,
}) => {
  const [activeMode, setActiveMode] = useState<GameMode>('rocket');
  const [showCodeViewer, setShowCodeViewer] = useState(false);
  const [copiedCode, setCopiedCode] = useState(false);

  // LocalStorage High Score System
  const [highScores, setHighScores] = useState<HighScores>(() => {
    try {
      const saved = localStorage.getItem('noise_game_high_scores');
      return saved ? JSON.parse(saved) : { rocket: 0, quiet: 0, target: 0, scream: 0 };
    } catch {
      return { rocket: 0, quiet: 0, target: 0, scream: 0 };
    }
  });

  const updateHighScore = (mode: GameMode, score: number) => {
    setHighScores((prev) => {
      if (score > prev[mode]) {
        const updated = { ...prev, [mode]: score };
        try {
          localStorage.setItem('noise_game_high_scores', JSON.stringify(updated));
        } catch (e) {
          console.error(e);
        }
        return updated;
      }
      return prev;
    });
  };

  // --- GAME 1: FLAPPY ROCKET (ROKET SUARA 60 FPS) ---
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const [rocketState, setRocketState] = useState<'idle' | 'playing' | 'gameover'>('idle');
  const [rocketScore, setRocketScore] = useState(0);

  const rocketPosRef = useRef({ y: 150, vy: 0 });
  const obstaclesRef = useRef<{ x: number; topHeight: number; bottomHeight: number; passed: boolean }[]>([]);
  const starsRef = useRef<{ x: number; y: number; collected: boolean }[]>([]);
  const animationFrameRef = useRef<number | null>(null);
  const currentDbRef = useRef(currentDb);
  currentDbRef.current = currentDb;

  const startRocketGame = () => {
    setRocketState('playing');
    setRocketScore(0);
    rocketPosRef.current = { y: 150, vy: 0 };
    obstaclesRef.current = [];
    starsRef.current = [];
  };

  useEffect(() => {
    if (activeMode !== 'rocket' || rocketState !== 'playing') {
      if (animationFrameRef.current) cancelAnimationFrame(animationFrameRef.current);
      return;
    }

    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    let frameCount = 0;
    let score = 0;

    const gameLoop = () => {
      frameCount++;
      const db = currentDbRef.current;

      // Fisika Dorongan Roket berdasarkan Suara Real-time
      const normDb = Math.max(30, Math.min(100, db));
      const thrust = (normDb - 45) * 0.15; // Suara > 45 dB menerbangkan roket naik!
      rocketPosRef.current.vy -= thrust * 0.15;
      rocketPosRef.current.vy += 0.35; // Gravitasi
      rocketPosRef.current.vy *= 0.92;

      rocketPosRef.current.y += rocketPosRef.current.vy;

      if (rocketPosRef.current.y < 20) {
        rocketPosRef.current.y = 20;
        rocketPosRef.current.vy = 0;
      }
      if (rocketPosRef.current.y > canvas.height - 20) {
        setRocketState('gameover');
        updateHighScore('rocket', score);
        return;
      }

      // Generator Rintangan Pilar Merah & Bintang Skor
      if (frameCount % 110 === 0) {
        const gap = 110;
        const minHeight = 30;
        const maxHeight = canvas.height - gap - minHeight;
        const topHeight = Math.floor(Math.random() * (maxHeight - minHeight)) + minHeight;
        obstaclesRef.current.push({
          x: canvas.width,
          topHeight,
          bottomHeight: canvas.height - topHeight - gap,
          passed: false,
        });

        starsRef.current.push({
          x: canvas.width + 20,
          y: topHeight + gap / 2,
          collected: false,
        });
      }

      // Pergerakan Rintangan
      for (let i = obstaclesRef.current.length - 1; i >= 0; i--) {
        const obs = obstaclesRef.current[i];
        obs.x -= 2.5;

        if (!obs.passed && obs.x < 60) {
          obs.passed = true;
          score += 10;
          setRocketScore(score);
        }

        // Deteksi Tabrakan (Collision Check)
        const rx = 60;
        const ry = rocketPosRef.current.y;
        if (obs.x < rx + 14 && obs.x + 40 > rx - 14) {
          if (ry - 12 < obs.topHeight || ry + 12 > canvas.height - obs.bottomHeight) {
            setRocketState('gameover');
            updateHighScore('rocket', score);
            return;
          }
        }

        if (obs.x < -50) obstaclesRef.current.splice(i, 1);
      }

      // Pengambilan Bintang
      for (let i = starsRef.current.length - 1; i >= 0; i--) {
        const star = starsRef.current[i];
        star.x -= 2.5;
        const dist = Math.hypot(star.x - 60, star.y - rocketPosRef.current.y);
        if (!star.collected && dist < 22) {
          star.collected = true;
          score += 25;
          setRocketScore(score);
        }
        if (star.x < -20) starsRef.current.splice(i, 1);
      }

      // RENDER CANVAS
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      const bgGrad = ctx.createLinearGradient(0, 0, 0, canvas.height);
      bgGrad.addColorStop(0, '#0f172a');
      bgGrad.addColorStop(1, '#020617');
      ctx.fillStyle = bgGrad;
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // Render Obstacles
      obstaclesRef.current.forEach((obs) => {
        ctx.fillStyle = '#f43f5e';
        ctx.fillRect(obs.x, 0, 40, obs.topHeight);
        ctx.fillRect(obs.x, canvas.height - obs.bottomHeight, 40, obs.bottomHeight);
      });

      // Render Bintang
      starsRef.current.forEach((star) => {
        if (!star.collected) {
          ctx.fillStyle = '#f59e0b';
          ctx.beginPath();
          ctx.arc(star.x, star.y, 8, 0, Math.PI * 2);
          ctx.fill();
        }
      });

      // Render Roket & Api Semburan
      const rx = 60;
      const ry = rocketPosRef.current.y;

      if (db > 50) {
        ctx.fillStyle = '#f97316';
        ctx.beginPath();
        ctx.arc(rx - 16, ry + (Math.random() * 6 - 3), 6 + Math.random() * 4, 0, Math.PI * 2);
        ctx.fill();
      }

      ctx.fillStyle = '#10b981';
      ctx.beginPath();
      ctx.ellipse(rx, ry, 18, 10, (rocketPosRef.current.vy * Math.PI) / 180, 0, Math.PI * 2);
      ctx.fill();

      // Overlap HUD Volume Desibel
      ctx.fillStyle = 'rgba(15, 23, 42, 0.7)';
      ctx.fillRect(10, 10, 160, 32);
      ctx.font = 'bold 12px monospace';
      ctx.fillStyle = db > 60 ? '#f43f5e' : db > 48 ? '#f59e0b' : '#10b981';
      ctx.fillText(`VOLUME: ${Math.round(db)} dB`, 20, 31);

      animationFrameRef.current = requestAnimationFrame(gameLoop);
    };

    animationFrameRef.current = requestAnimationFrame(gameLoop);
    return () => {
      if (animationFrameRef.current) cancelAnimationFrame(animationFrameRef.current);
    };
  }, [activeMode, rocketState, updateHighScore]);

  // --- GAME 2: MASTER HENING ---
  const [quietState, setQuietState] = useState<'idle' | 'playing' | 'success' | 'failed'>('idle');
  const [quietTimeLeft, setQuietTimeLeft] = useState(20);
  const [quietMaxDbReached, setQuietMaxDbReached] = useState(30);

  const startQuietGame = () => {
    setQuietState('playing');
    setQuietTimeLeft(20);
    setQuietMaxDbReached(currentDb);
  };

  useEffect(() => {
    if (activeMode !== 'quiet' || quietState !== 'playing') return;

    if (currentDb > quietMaxDbReached) setQuietMaxDbReached(currentDb);

    if (currentDb > 55) {
      setQuietState('failed');
      return;
    }

    const timer = setInterval(() => {
      setQuietTimeLeft((prev) => {
        if (prev <= 1) {
          clearInterval(timer);
          setQuietState('success');
          updateHighScore('quiet', 100);
          return 0;
        }
        return prev - 1;
      });
    }, 1000);

    return () => clearInterval(timer);
  }, [activeMode, quietState, currentDb, quietMaxDbReached, updateHighScore]);

  // --- GAME 3: TEMBAK DESIBEL PAS ---
  const [targetDb, setTargetDb] = useState(65);
  const [targetState, setTargetState] = useState<'idle' | 'playing' | 'result'>('idle');
  const [targetTimer, setTargetTimer] = useState(5);
  const [targetPrecisionScore, setTargetPrecisionScore] = useState(0);

  const startTargetGame = () => {
    const randomTarget = Math.floor(Math.random() * 35) + 50;
    setTargetDb(randomTarget);
    setTargetState('playing');
    setTargetTimer(5);
  };

  useEffect(() => {
    if (activeMode !== 'target' || targetState !== 'playing') return;

    const timer = setInterval(() => {
      setTargetTimer((prev) => {
        if (prev <= 1) {
          clearInterval(timer);
          const diff = Math.abs(currentDb - targetDb);
          const score = Math.max(0, Math.round(100 - diff * 3.5));
          setTargetPrecisionScore(score);
          setTargetState('result');
          updateHighScore('target', score);
          return 0;
        }
        return prev - 1;
      });
    }, 1000);

    return () => clearInterval(timer);
  }, [activeMode, targetState, currentDb, targetDb, updateHighScore]);

  // --- GAME 4: SCREAM POWER METER ---
  const [screamState, setScreamState] = useState<'idle' | 'measuring' | 'result'>('idle');
  const [screamCountdown, setScreamCountdown] = useState(3);
  const [screamPeakDb, setScreamPeakDb] = useState(0);

  const startScreamGame = () => {
    setScreamState('measuring');
    setScreamCountdown(3);
    setScreamPeakDb(0);
  };

  useEffect(() => {
    if (activeMode !== 'scream' || screamState !== 'measuring') return;

    if (currentDb > screamPeakDb) setScreamPeakDb(currentDb);

    const timer = setInterval(() => {
      setScreamCountdown((prev) => {
        if (prev <= 1) {
          clearInterval(timer);
          setScreamState('result');
          updateHighScore('scream', Math.round(screamPeakDb));
          return 0;
        }
        return prev - 1;
      });
    }, 1000);

    return () => clearInterval(timer);
  }, [activeMode, screamState, currentDb, screamPeakDb, updateHighScore]);

  return (
    <div className="space-y-6">
      {/* Tab Navigasi Minigame & Canvas UI disajikan di tampilan live */}
    </div>
  );
};
