import React, { useState, useEffect, useRef, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, doc, setDoc, getDoc, updateDoc, onSnapshot
} from 'firebase/firestore';
import { 
  getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged 
} from 'firebase/auth';
import { Edit2, Check, X } from 'lucide-react';

// --- CONFIG ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'gator-500-pro';

const SKINS = [
  { id: 'classic', name: 'Swamp Green', color: '#2d5a27', price: 0 },
  { id: 'albino', name: 'Albino White', color: '#f0f0f0', price: 50 },
  { id: 'neon', name: 'Cyber Neon', color: '#00ffcc', price: 150 },
  { id: 'lava', name: 'Magma Scale', color: '#ff4500', price: 300 },
  { id: 'rainbow', name: 'Chroma Glow', color: 'rainbow', price: 600 },
  { id: 'gold', name: 'Golden King', color: '#ffd700', price: 1000 },
];

const AI_NAMES = ["Swamp Runner", "Bayou Blur", "Croc Speed", "Marsh King", "Mud Slinger", "Scale Racer", "Tail Whip"];

const TRACK_POINTS = [
  { x: 0, z: 0 }, { x: 500, z: -200 }, { x: 1200, z: 200 }, { x: 1500, z: 1000 },
  { x: 1000, z: 1800 }, { x: 200, z: 2000 }, { x: -800, z: 1500 }, { x: -1400, z: 800 },
  { x: -800, z: 0 }, { x: 0, z: 0 }
];

const PANEL = "pointer-events-auto bg-black/60 backdrop-blur-3xl border border-white/10 p-8 rounded-[3rem] shadow-2xl relative z-10";
const BUTTON = "px-6 py-4 bg-lime-600 hover:bg-lime-500 transition-all rounded-2xl font-black uppercase tracking-widest active:scale-95 disabled:opacity-50 shadow-lg shadow-lime-900/40 cursor-pointer";

const XP_FOR_OVR_UP = 500;
const getOvrColor = (ovr) => {
    if(ovr >= 90) return '#fbbf24'; 
    if(ovr >= 80) return '#a855f7'; 
    if(ovr >= 70) return '#3b82f6'; 
    if(ovr >= 60) return '#22c55e'; 
    return '#9ca3af'; 
};

const LobbyBackground = () => (
  <div className="absolute inset-0 overflow-hidden pointer-events-none bg-[#020a05]">
    <style>{`
      @keyframes drift {
        0% { transform: translateY(0) translateX(0); opacity: 0; }
        20% { opacity: 0.4; }
        80% { opacity: 0.4; }
        100% { transform: translateY(-100vh) translateX(50px); opacity: 0; }
      }
    `}</style>
    {[...Array(50)].map((_, i) => (
      <div 
        key={i}
        className="absolute bg-white rounded-full"
        style={{
          width: Math.random() * 3 + 1 + 'px',
          height: Math.random() * 3 + 1 + 'px',
          left: Math.random() * 100 + '%',
          top: (Math.random() * 100 + 100) + '%',
          animation: `drift ${Math.random() * 10 + 10}s linear infinite`,
          animationDelay: `-${Math.random() * 10}s`
        }}
      />
    ))}
    <div className="absolute inset-0 bg-gradient-to-t from-emerald-950/30 to-transparent" />
  </div>
);

export default function App() {
  const [user, setUser] = useState(null);
  const [profile, setProfile] = useState(null);
  const [gameState, setGameState] = useState('menu'); 
  const [countdown, setCountdown] = useState(null);
  const [boostLevel, setBoostLevel] = useState(100);
  const [boostCooldown, setBoostCooldown] = useState(false);
  const [raceResults, setRaceResults] = useState(null);
  const [raceFinished, setRaceFinished] = useState(false); 
  const [standings, setStandings] = useState([]);
  const [aiStates, setAiStates] = useState([]);
  const [isEditingName, setIsEditingName] = useState(false);
  const [tempName, setTempName] = useState("");
  
  const mountRef = useRef(null);
  const stateRef = useRef({ 
    isStarted: false, speed: 0, rotation: 1.1,
    keys: {}, progress: 0, playerPos: { x: 0, z: 0 },
    boostActive: false, boostCooldownTimer: 0,
    boostCooldownActive: false,
    requestId: null
  });

  const engineRef = useRef({
    scene: null, camera: null, renderer: null, player: null, 
    clock: null, ai: [], curve: null, visual: null,
    tail: null, jaw: null, feet: [], camSmoothPos: null,
    allLabels: []
  });

  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    return onAuthStateChanged(auth, (u) => {
      if (u) {
        setUser(u);
        const docRef = doc(db, 'artifacts', appId, 'users', u.uid, 'profile', 'data');
        onSnapshot(docRef, (snap) => {
          if (snap.exists()) {
            setProfile(snap.data());
          } else {
            const p = { 
              username: `Gator_${u.uid.slice(0, 4)}`, 
              trophies: 0, 
              xp: 0, 
              ovr: 60, 
              winStreak: 0, 
              unlockedSkins: ['classic'], 
              activeSkin: 'classic' 
            };
            setDoc(docRef, p);
          }
        });
      }
    });
  }, []);

  useEffect(() => {
    const down = (e) => stateRef.current.keys[e.key.toLowerCase()] = true;
    const up = (e) => stateRef.current.keys[e.key.toLowerCase()] = false;
    window.addEventListener('keydown', down);
    window.addEventListener('keyup', up);
    return () => { 
      window.removeEventListener('keydown', down); 
      window.removeEventListener('keyup', up); 
    };
  }, []);

  const updateUsername = async () => {
    if (!tempName.trim()) return;
    const docRef = doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'data');
    await updateDoc(docRef, { username: tempName.trim() });
    setIsEditingName(false);
  };

  const createGatorModel = (color, name, ovr, streak, THREE) => {
    const group = new THREE.Group();
    const visual = new THREE.Group();
    let mat;
    if (color === 'rainbow') {
      mat = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0, metalness: 0.5, flatShading: true });
      mat.userData.isRainbow = true;
    } else {
      mat = new THREE.MeshStandardMaterial({ color, roughness: 0.2, flatShading: true }); 
    }
    group.scale.set(3, 3, 3);

    const body = new THREE.Mesh(new THREE.BoxGeometry(3, 1.4, 5), mat);
    body.position.y = 0.7;
    visual.add(body);

    const feet = [];
    const footGeo = new THREE.BoxGeometry(0.8, 0.8, 1.2);
    const footPos = [{ x: 1.8, z: 2 }, { x: -1.8, z: 2 }, { x: 1.8, z: -2 }, { x: -1.8, z: -2 }];
    footPos.forEach(p => {
        const foot = new THREE.Mesh(footGeo, mat);
        foot.position.set(p.x, 0.4, p.z);
        visual.add(foot);
        feet.push(foot);
    });

    const tailGroup = new THREE.Group();
    tailGroup.position.set(0, 0.7, -2.5);
    const t1 = new THREE.Mesh(new THREE.BoxGeometry(2, 1, 3), mat); t1.position.z = -1.5; tailGroup.add(t1);
    const t2 = new THREE.Mesh(new THREE.BoxGeometry(1.2, 0.7, 3), mat); t2.position.z = -4; tailGroup.add(t2);
    visual.add(tailGroup);

    const headGroup = new THREE.Group();
    headGroup.position.set(0, 0.8, 2.5);
    const upperJaw = new THREE.Mesh(new THREE.BoxGeometry(2.2, 0.6, 4), mat); upperJaw.position.z = 2; headGroup.add(upperJaw);
    const lowerJaw = new THREE.Mesh(new THREE.BoxGeometry(2, 0.4, 3.8), mat); lowerJaw.position.set(0, -0.4, 1.9); headGroup.add(lowerJaw);
    visual.add(headGroup);

    const eyeMat = new THREE.MeshBasicMaterial({ color: 0xff3300 });
    const e1 = new THREE.Mesh(new THREE.SphereGeometry(0.25, 8, 8), eyeMat); e1.position.set(0.7, 0.4, 1.2); upperJaw.add(e1);
    const e2 = e1.clone(); e2.position.set(-0.7, 0.4, 1.2); upperJaw.add(e2);

    const canvas = document.createElement('canvas');
    canvas.width = 512; canvas.height = 180;
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = 'rgba(0,0,0,0.6)';
    ctx.roundRect(0, 40, 512, 128, 64); ctx.fill();
    if (streak > 0) {
        ctx.font = 'bold 36px Arial';
        ctx.fillStyle = '#ff4500';
        ctx.textAlign = 'center';
        ctx.fillText(`🔥 ${streak}`, 256, 35);
    }
    ctx.font = 'bold 60px Arial';
    ctx.textAlign = 'center';
    ctx.fillStyle = getOvrColor(ovr);
    ctx.fillText(`${ovr}`, 70, 125);
    ctx.fillStyle = 'white';
    ctx.fillText(name, 280, 125);

    const tex = new THREE.CanvasTexture(canvas);
    const label = new THREE.Mesh(new THREE.PlaneGeometry(10, 3.5), new THREE.MeshBasicMaterial({ map: tex, transparent: true, side: THREE.DoubleSide }));
    label.position.y = 6.0;
    group.add(label);
    
    group.add(visual);
    return { group, visual, tail: tailGroup, jaw: headGroup, feet, mat, label };
  };

  const startRace = () => {
    setGameState('racing');
    setRaceFinished(false);
    setBoostLevel(100);
    setBoostCooldown(false);
    stateRef.current.progress = 0;
    stateRef.current.speed = 0;
    stateRef.current.rotation = 1.1; 
    setTimeout(() => { if (mountRef.current) initThree(); }, 100);
  };

  const initThree = () => {
    const THREE = window.THREE;
    const scene = new THREE.Scene();
    scene.background = new THREE.Color('#010804');
    scene.fog = new THREE.Fog('#010804', 500, 5000);

    const camera = new THREE.PerspectiveCamera(75, mountRef.current.clientWidth / mountRef.current.clientHeight, 1, 20000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(mountRef.current.clientWidth, mountRef.current.clientHeight);
    mountRef.current.appendChild(renderer.domElement);

    scene.add(new THREE.AmbientLight(0xffffff, 0.4)); 
    const sun = new THREE.DirectionalLight(0x22c55e, 1.0);
    sun.position.set(1000, 2000, 1000);
    scene.add(sun);
    
    const curve = new THREE.CatmullRomCurve3(TRACK_POINTS.map(p => new THREE.Vector3(p.x, 0, p.z)), true);
    curve.updateArcLengths();

    const trackGeo = new THREE.TubeGeometry(curve, 400, 120, 20, true);
    const trackMesh = new THREE.Mesh(trackGeo, new THREE.MeshStandardMaterial({ 
        color: '#064e3b', 
        side: THREE.BackSide,
        emissive: '#065f46',
        emissiveIntensity: 0.4
    }));
    scene.add(trackMesh);

    const wallGeo = new THREE.TubeGeometry(curve, 400, 132, 20, true);
    const wallMesh = new THREE.Mesh(wallGeo, new THREE.MeshStandardMaterial({ 
        color: '#10b981', 
        transparent: true, 
        opacity: 0.3,
        side: THREE.BackSide,
        metalness: 0.8,
        roughness: 0,
        emissive: '#059669',
        emissiveIntensity: 1.5
    }));
    scene.add(wallMesh);

    const getPos = (prog, lane) => {
      const t = (prog < 0 ? 1 + (prog % 1) : prog % 1);
      const p = curve.getPointAt(t);
      const tan = curve.getTangentAt(t);
      const n = new THREE.Vector3(-tan.z, 0, tan.x).normalize();
      return p.clone().add(n.multiplyScalar(lane));
    };

    const labels = [];
    const pSkin = SKINS.find(s => s.id === profile.activeSkin) || SKINS[0];
    const pObj = createGatorModel(pSkin.color, profile.username, profile.ovr, profile.winStreak || 0, THREE);
    pObj.group.position.copy(getPos(0.01, 30)); 
    pObj.group.rotation.y = 1.1;
    scene.add(pObj.group);
    labels.push(pObj.label);

    const bots = [];
    for(let i=0; i<7; i++) {
      const skin = SKINS[i % SKINS.length];
      const aiOvr = Math.max(40, Math.min(99, profile.ovr + Math.floor(Math.random() * 10) - 4));
      const aiStreak = Math.floor(Math.random() * 5); 
      const name = AI_NAMES[i % AI_NAMES.length];
      const bObj = createGatorModel(skin.color, name, aiOvr, aiStreak, THREE);
      const lane = (i % 2 === 0) ? -40 - (i*15) : -80 - (i*15); 
      const startProg = 0.005 - (i * 0.015);
      bObj.group.position.copy(getPos(startProg, lane));
      scene.add(bObj.group);
      labels.push(bObj.label);
      
      bots.push({ 
        ...bObj, 
        name,
        prog: startProg, lane, ovr: aiOvr, winStreak: aiStreak,
        color: skin.color === 'rainbow' ? '#00ff00' : skin.color,
        baseSpeed: 0.018 + (aiOvr * 0.0001), currentSpeed: 0, swaySpeed: 0.6 + Math.random(),
        boostActive: false, boostCooldown: 0
      });
    }

    engineRef.current = { 
      scene, camera, renderer, player: pObj.group, visual: pObj.visual, 
      tail: pObj.tail, jaw: pObj.jaw, feet: pObj.feet, clock: new THREE.Clock(), ai: bots, curve,
      camSmoothPos: pObj.group.position.clone().add(new THREE.Vector3(0, 100, -200)),
      allLabels: labels
    };
    
    setCountdown(3);
    const countdownInterval = setInterval(() => {
      setCountdown(c => {
        if (c === 1) { 
            clearInterval(countdownInterval); 
            stateRef.current.isStarted = true; 
            setTimeout(() => setCountdown(null), 1000); 
            return "GO!"; 
        }
        return typeof c === 'number' ? c - 1 : c;
      });
    }, 1000);

    stateRef.current.requestId = requestAnimationFrame(animate);
  };

  const finishRace = async (position) => {
    if (raceFinished) return;
    setRaceFinished(true);
    stateRef.current.isStarted = false;
    let rewardTrophies = position === 1 ? 30 : position === 2 ? 15 : position === 3 ? 5 : 0;
    let rewardXp = position === 1 ? 250 : position === 2 ? 150 : position === 3 ? 100 : 50; 
    
    let newXp = (profile.xp || 0) + rewardXp;
    let newOvr = profile.ovr || 60;
    while (newXp >= XP_FOR_OVR_UP) {
        newXp -= XP_FOR_OVR_UP;
        newOvr = Math.min(99, newOvr + 1);
    }

    let newStreak = position === 1 ? (profile.winStreak || 0) + 1 : 0;

    const updates = { 
      trophies: (profile.trophies || 0) + rewardTrophies, 
      xp: newXp, 
      ovr: newOvr, 
      winStreak: newStreak 
    };
    
    setRaceResults({ position, rewardTrophies, rewardXp });
    await updateDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'data'), updates);
  };

  const animate = () => {
    if (!engineRef.current.renderer) return;
    stateRef.current.requestId = requestAnimationFrame(animate);
    const delta = Math.min(engineRef.current.clock.getDelta(), 0.1);
    const elapsed = engineRef.current.clock.getElapsedTime();
    const { keys, isStarted } = stateRef.current;
    const THREE = window.THREE;

    if (isStarted && !raceFinished) {
      const performanceMult = 1 + ((profile.ovr - 50) * 0.01); 
      let wantsBoost = (keys['shift'] || keys[' ']) && boostLevel > 1 && !stateRef.current.boostCooldownActive;
      
      if (wantsBoost) {
          stateRef.current.boostActive = true;
          setBoostLevel(b => {
            const next = b - 35 * delta; 
            if (next <= 0) {
              stateRef.current.boostCooldownActive = true;
              stateRef.current.boostCooldownTimer = 4.0; 
              setBoostCooldown(true);
            }
            return Math.max(0, next);
          });
      } else {
          stateRef.current.boostActive = false;
          if (stateRef.current.boostCooldownActive) {
              stateRef.current.boostCooldownTimer -= delta;
              if (stateRef.current.boostCooldownTimer <= 0) {
                  stateRef.current.boostCooldownActive = false;
                  setBoostCooldown(false);
              }
          }
          setBoostLevel(b => Math.min(100, b + 15 * delta)); 
      }

      let accelForce = 0;
      if (keys['w'] || keys['arrowup']) accelForce = (stateRef.current.boostActive ? 2200 : 850) * performanceMult; 
      if (keys['s'] || keys['arrowdown']) accelForce = -500 * performanceMult;
      
      stateRef.current.speed += accelForce * delta;
      stateRef.current.speed *= 0.97; 
      
      let steer = 0;
      if (keys['a'] || keys['arrowleft']) steer = 2.8; 
      if (keys['d'] || keys['arrowright']) steer = -2.8;
      
      stateRef.current.rotation += steer * delta * (Math.abs(stateRef.current.speed) / 350);
      engineRef.current.player.rotation.y = stateRef.current.rotation;

      const moveVec = new THREE.Vector3(0, 0, stateRef.current.speed * delta);
      moveVec.applyQuaternion(engineRef.current.player.quaternion);
      engineRef.current.player.position.add(moveVec);

      const t = stateRef.current.progress % 1;
      const point = engineRef.current.curve.getPointAt(t < 0 ? 1 + t : t);
      if (engineRef.current.player.position.distanceTo(point) > 115) {
          stateRef.current.speed *= 0.8;
          const pushBack = point.clone().sub(engineRef.current.player.position).normalize();
          engineRef.current.player.position.add(pushBack.multiplyScalar(3));
      }

      stateRef.current.progress += (stateRef.current.speed * delta) / 6000;
      
      // Calculate real-time standings
      const currentStandings = [
        { id: 'player', name: profile.username, progress: stateRef.current.progress, ovr: profile.ovr, streak: profile.winStreak || 0 },
        ...engineRef.current.ai.map((a, idx) => ({ id: `ai-${idx}`, name: a.name, progress: a.prog, ovr: a.ovr, streak: a.winStreak }))
      ].sort((a,b) => b.progress - a.progress);
      setStandings(currentStandings);

      if (stateRef.current.progress >= 3.0) {
        const finalPos = currentStandings.findIndex(s => s.id === 'player') + 1;
        finishRace(finalPos);
      }

      const pMoveFactor = Math.abs(stateRef.current.speed) / 300;
      engineRef.current.tail.rotation.y = Math.sin(elapsed * 15) * (0.3 + pMoveFactor);
      engineRef.current.jaw.rotation.x = THREE.MathUtils.lerp(engineRef.current.jaw.rotation.x, stateRef.current.boostActive ? -0.8 : 0, 0.2);
      engineRef.current.feet.forEach((f, i) => f.rotation.x = Math.sin(elapsed * 15 * (1 + pMoveFactor) + (i % 2 === 0 ? 0 : Math.PI)) * 0.5);

      const nextAiStates = [];
      engineRef.current.ai.forEach((b, i) => {
        if (!b.boostActive && b.boostCooldown <= 0) {
            const playerProgress = stateRef.current.progress;
            const boostChance = b.prog < playerProgress ? 0.01 : 0.002;
            if (Math.random() < boostChance) {
                b.boostActive = true;
                b.boostTime = 2.0 + Math.random() * 2;
            }
        }
        
        if (b.boostActive) {
            b.boostTime -= delta;
            if (b.boostTime <= 0) {
                b.boostActive = false;
                b.boostCooldown = 4.0 + Math.random() * 3;
            }
        } else if (b.boostCooldown > 0) {
            b.boostCooldown -= delta;
        }

        const targetSpeed = b.boostActive ? b.baseSpeed * 1.8 : b.baseSpeed;
        b.currentSpeed = THREE.MathUtils.lerp(b.currentSpeed, targetSpeed, 0.05);
        b.prog += b.currentSpeed * delta;

        const bt = b.prog % 1;
        const p = engineRef.current.curve.getPointAt(bt < 0 ? 1 + bt : bt);
        const tan = engineRef.current.curve.getTangentAt(bt < 0 ? 1 + bt : bt);
        const n = new THREE.Vector3(-tan.z, 0, tan.x).normalize();
        let lateralOffset = b.lane + Math.sin(elapsed * b.swaySpeed + i) * 55;
        lateralOffset = THREE.MathUtils.clamp(lateralOffset, -110, 110);
        b.group.position.lerp(p.clone().add(n.multiplyScalar(lateralOffset)), 0.1);
        b.group.lookAt(b.group.position.clone().add(tan));
        
        const aiMoveFactor = b.currentSpeed / 0.02; 
        b.tail.rotation.y = Math.sin(elapsed * 15) * (0.3 + aiMoveFactor);
        b.jaw.rotation.x = THREE.MathUtils.lerp(b.jaw.rotation.x, b.boostActive ? -0.8 : 0, 0.1);
        b.feet.forEach((f, fi) => f.rotation.x = Math.sin(elapsed * 15 * (1 + aiMoveFactor) + (fi % 2 === 0 ? 0 : Math.PI)) * 0.5);
        nextAiStates.push({ color: b.color, pos: { x: b.group.position.x, z: b.group.position.z } });
      });

      setAiStates(nextAiStates);
      stateRef.current.playerPos = { x: engineRef.current.player.position.x, z: engineRef.current.player.position.z };
    }

    const camOffset = new THREE.Vector3(0, 65, -130).applyQuaternion(engineRef.current.player.quaternion);
    engineRef.current.camSmoothPos.lerp(engineRef.current.player.position.clone().add(camOffset), 0.1); 
    engineRef.current.camera.position.copy(engineRef.current.camSmoothPos);
    engineRef.current.camera.lookAt(engineRef.current.player.position.clone().add(new THREE.Vector3(0, 15, 50).applyQuaternion(engineRef.current.player.quaternion)));
    
    engineRef.current.allLabels.forEach(label => {
        if (label) label.lookAt(engineRef.current.camera.position);
    });

    engineRef.current.renderer.render(engineRef.current.scene, engineRef.current.camera);
  };

  const handleReturnMenu = () => {
    cancelAnimationFrame(stateRef.current.requestId);
    if (engineRef.current.renderer) {
      engineRef.current.renderer.dispose();
      if (mountRef.current && mountRef.current.contains(engineRef.current.renderer.domElement)) {
        mountRef.current.removeChild(engineRef.current.renderer.domElement);
      }
    }
    setGameState('menu');
    setRaceFinished(false);
    setRaceResults(null);
  };

  const radarTrackData = useMemo(() => {
    const minX = -1500, maxX = 1600, minZ = -300, maxZ = 2100;
    const scale = (val, min, max) => ((val - min) / (max - min)) * 140 + 20; 
    return TRACK_POINTS.map(p => ({ x: scale(p.x, minX, maxX), y: scale(p.z, minZ, maxZ) }));
  }, []);

  const scaleCoord = (val, min, max) => ((val - min) / (max - min)) * 140 + 20;

  if (!profile) return <div className="bg-black h-screen flex items-center justify-center text-lime-400 font-black">🐊 LOADING...</div>;

  return (
    <div className="w-full h-screen bg-black overflow-hidden text-white font-sans select-none relative">
      <div ref={mountRef} className="w-full h-full" style={{ display: gameState === 'racing' ? 'block' : 'none' }} />

      {gameState === 'menu' && (
        <div className="absolute inset-0 flex items-center justify-center">
          <LobbyBackground />
          <div className={`${PANEL} w-[550px] text-center space-y-10`}>
            <div className="space-y-2">
                <h1 className="text-8xl font-black italic text-lime-400 tracking-tighter drop-shadow-[0_0_30px_rgba(163,230,53,0.5)] uppercase">Gator 500</h1>
                <p className="text-xs uppercase tracking-[0.5em] font-black text-lime-200/40">The Professional Racing Gator</p>
            </div>
            
            <div className="bg-white/5 p-8 rounded-[2.5rem] relative border border-white/5 overflow-hidden">
                <div className="absolute inset-0 bg-gradient-to-br from-lime-500/5 to-transparent" />
                <div className="relative z-10 flex flex-col items-center">
                    <div className="flex items-center gap-3">
                        <span className="text-4xl font-black" style={{ color: getOvrColor(profile.ovr) }}>{profile.ovr}</span>
                        {isEditingName ? (
                          <div className="flex items-center gap-2">
                            <input 
                              autoFocus
                              value={tempName}
                              onChange={e => setTempName(e.target.value)}
                              className="bg-black/50 border border-white/20 rounded-xl px-4 py-2 outline-none text-2xl font-bold w-48"
                            />
                            <button onClick={updateUsername} className="text-lime-400"><Check size={24}/></button>
                            <button onClick={() => setIsEditingName(false)} className="text-red-400"><X size={24}/></button>
                          </div>
                        ) : (
                          <div className="flex items-center gap-2 group">
                            <p className="text-4xl font-bold uppercase tracking-tighter">{profile.username}</p>
                            <Edit2 
                              size={20} 
                              className="opacity-30 group-hover:opacity-100 cursor-pointer transition-opacity" 
                              onClick={() => {
                                setTempName(profile.username);
                                setIsEditingName(true);
                              }}
                            />
                          </div>
                        )}
                    </div>
                    <div className="mt-4 w-full h-2 bg-white/10 rounded-full overflow-hidden">
                       <div className="h-full bg-lime-400 shadow-[0_0_10px_#a3e635]" style={{ width: `${(profile.xp / XP_FOR_OVR_UP) * 100}%` }} />
                    </div>
                    <p className="text-[10px] mt-2 font-black uppercase text-white/40 tracking-widest">Progress to {profile.ovr + 1} OVR</p>
                    <div className="mt-4 flex justify-center gap-6">
                       <span className="bg-lime-600/20 text-lime-400 px-4 py-1 rounded-full text-xs font-black uppercase border border-lime-500/20">🏆 {profile.trophies}</span>
                       <span className="bg-orange-600/20 text-orange-400 px-4 py-1 rounded-full text-xs font-black uppercase border border-orange-500/20">🔥 Streak: {profile.winStreak || 0}</span>
                    </div>
                </div>
            </div>

            <button onClick={startRace} className={BUTTON}>Start Race</button>
            <button onClick={() => setGameState('shop')} className="block mx-auto text-xs opacity-50 font-black uppercase tracking-widest hover:opacity-100 transition-opacity cursor-pointer">Skins & Garage</button>
          </div>
        </div>
      )}

      {gameState === 'shop' && (
        <div className="absolute inset-0 flex items-center justify-center bg-black/95 z-[300]">
          <LobbyBackground />
          <div className={`${PANEL} w-[600px]`}>
            <div className="flex justify-between items-center mb-6">
                <h2 className="text-4xl font-black italic uppercase text-lime-400">The Garage</h2>
                <span className="text-xl font-black text-white/50">🏆 {profile.trophies}</span>
            </div>
            <div className="grid grid-cols-2 gap-4">
              {SKINS.map(s => (
                <button key={s.id} onClick={() => {
                  if (profile.unlockedSkins.includes(s.id)) {
                    updateDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'data'), { activeSkin: s.id });
                  } else if (profile.trophies >= s.price) {
                    const newUnlocked = [...profile.unlockedSkins, s.id];
                    const newTrophies = profile.trophies - s.price;
                    updateDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'data'), { unlockedSkins: newUnlocked, trophies: newTrophies, activeSkin: s.id });
                  }
                }} 
                  className={`p-6 rounded-3xl border transition-all cursor-pointer ${profile.activeSkin === s.id ? 'border-lime-500 bg-lime-500/10' : 'border-white/5 bg-white/5'}`}>
                    <div className="w-full h-8 rounded-full mb-3" style={{ background: s.color === 'rainbow' ? 'linear-gradient(45deg, red, blue, green, yellow, purple)' : s.color }} />
                    <span className="font-black uppercase tracking-widest text-[10px] block">{s.name}</span>
                    <span className="text-[10px] font-black text-lime-400">{profile.unlockedSkins.includes(s.id) ? 'Owned' : `🏆 ${s.price}`}</span>
                </button>
              ))}
            </div>
            <button onClick={() => setGameState('menu')} className="w-full mt-6 p-4 bg-white/5 rounded-xl font-bold uppercase cursor-pointer">Back</button>
          </div>
        </div>
      )}

      {countdown && (
        <div className="absolute inset-0 flex items-center justify-center pointer-events-none z-[150]">
          <span className="text-[20rem] font-black italic text-lime-400 drop-shadow-[0_0_50px_rgba(163,230,53,0.8)]">{countdown}</span>
        </div>
      )}

      {raceFinished && (
        <div className="absolute inset-0 flex items-center justify-center bg-black/80 backdrop-blur-xl z-[400]">
           <div className={`${PANEL} text-center space-y-6`}>
              <h2 className="text-8xl font-black italic text-lime-400">#{raceResults?.position}</h2>
              <p className="text-xl font-black uppercase tracking-widest">Rewards: +{raceResults?.rewardTrophies}🏆 +{raceResults?.rewardXp}XP</p>
              {raceResults?.position === 1 && <p className="text-orange-400 font-black uppercase">Win Streak Increased! 🔥</p>}
              <button onClick={handleReturnMenu} className={BUTTON}>Return to Menu</button>
           </div>
        </div>
      )}

      {gameState === 'racing' && (
        <div className="absolute inset-0 pointer-events-none p-10 flex flex-col justify-between z-[50]">
          <div className="flex justify-between items-start">
            <div className="bg-black/80 border border-white/10 p-6 rounded-[2.5rem] backdrop-blur-xl w-72">
                <div className="flex justify-between items-center mb-4">
                  <h3 className="text-[10px] font-black uppercase tracking-[0.3em] text-white/40">Leaderboard</h3>
                  <p className="text-xs font-black text-lime-400">LAP {Math.min(3, Math.floor(stateRef.current.progress) + 1)}/3</p>
                </div>
                <div className="space-y-2">
                    {standings.slice(0, 8).map((s, i) => (
                        <div key={s.id} className={`flex items-center justify-between ${s.id === 'player' ? 'bg-lime-500/10 p-1.5 -mx-2 px-3 rounded-xl border border-lime-500/20' : 'opacity-80'}`}>
                            <div className="flex items-center gap-3">
                                <span className="text-xs font-black text-white/30">{i + 1}</span>
                                <span className={`text-[11px] font-bold uppercase truncate w-24 ${s.id === 'player' ? 'text-lime-400' : 'text-white'}`}>{s.name}</span>
                                {s.streak > 0 && <span className="text-[9px] text-orange-400 font-black">🔥{s.streak}</span>}
                            </div>
                            <span className="text-[10px] font-black px-2 py-0.5 rounded-md min-w-[28px] text-center" style={{ backgroundColor: `${getOvrColor(s.ovr)}`, color: 'black' }}>{s.ovr}</span>
                        </div>
                    ))}
                </div>
            </div>

            <div className="w-48 h-48 bg-black/80 rounded-full border-4 border-lime-500/20 backdrop-blur-md relative overflow-hidden">
                <svg className="absolute inset-0 w-full h-full" viewBox="0 0 180 180">
                    <path d={`M ${radarTrackData.map(p => `${p.x},${p.y}`).join(' L ')} Z`} fill="none" stroke="white" strokeWidth="2" opacity="0.3" />
                </svg>
                {aiStates.map((ai, i) => (
                    <div key={i} className="absolute w-2 h-2 rounded-full -translate-x-1/2 -translate-y-1/2" 
                          style={{ 
                              left: `${scaleCoord(ai.pos.x, -1500, 1600)}px`, 
                              top: `${scaleCoord(ai.pos.z, -300, 2100)}px`, 
                              backgroundColor: ai.color 
                          }} />
                ))}
                <div className="absolute w-3 h-3 bg-white rounded-full -translate-x-1/2 -translate-y-1/2 shadow-[0_0_12px_white]" 
                      style={{ 
                          left: `${scaleCoord(stateRef.current.playerPos.x, -1500, 1600)}px`, 
                          top: `${scaleCoord(stateRef.current.playerPos.z, -300, 2100)}px` 
                      }} />
            </div>
          </div>
          
          <div className="flex justify-between items-end">
             <div className="bg-black/90 p-10 rounded-full border-8 border-lime-500/10 w-60 h-60 flex flex-col items-center justify-center shadow-[0_0_40px_rgba(163,230,53,0.1)]">
                <p className="text-8xl font-black italic tracking-tighter text-white">{Math.floor(Math.abs(stateRef.current.speed) * 0.15)}</p>
                <p className="text-xs text-lime-400 font-black uppercase tracking-[0.4em]">Knots</p>
             </div>
             
             <div className="w-28 h-80 bg-black/90 rounded-[4rem] border-8 border-white/5 overflow-hidden relative">
                <div className={`absolute bottom-0 w-full transition-all duration-300 ${boostCooldown ? 'bg-red-500 animate-pulse' : stateRef.current.boostActive ? 'bg-white' : 'bg-lime-500'}`} style={{ height: `${boostLevel}%` }} />
                <div className="absolute inset-0 flex items-center justify-center pointer-events-none">
                    <p className="text-lg font-black rotate-90 tracking-[1em] text-white/20 uppercase">
                      {boostCooldown ? "Cooldown" : "Nitro"}
                    </p>
                </div>
             </div>
          </div>
        </div>
      )}
    </div>
  );
}
