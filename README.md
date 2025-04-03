
// Firebase config (Make sure you replace with your own Firebase credentials) import { initializeApp } from "firebase/app"; import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword } from "firebase/auth"; import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, getDocs, where, updateDoc, doc } from "firebase/firestore"; import { useEffect, useState } from "react"; import { Button } from "@/components/ui/button"; import { Card, CardContent } from "@/components/ui/card"; import { motion } from "framer-motion";

const firebaseConfig = { apiKey: "YOUR_API_KEY", authDomain: "YOUR_AUTH_DOMAIN", projectId: "YOUR_PROJECT_ID", storageBucket: "YOUR_STORAGE_BUCKET", messagingSenderId: "YOUR_SENDER_ID", appId: "YOUR_APP_ID", };

const app = initializeApp(firebaseConfig); const auth = getAuth(app); const db = getFirestore(app);

export default function AviatorGame() { const [user, setUser] = useState(null); const [email, setEmail] = useState(""); const [password, setPassword] = useState(""); const [isLogin, setIsLogin] = useState(true); const [position, setPosition] = useState({ x: 0, y: 0 }); const [angle, setAngle] = useState(0); const [gameStarted, setGameStarted] = useState(false); const [isAdmin, setIsAdmin] = useState(false); const [betAmount, setBetAmount] = useState(""); const [betPlaced, setBetPlaced] = useState(false); const [scoreboard, setScoreboard] = useState([]);

const handleAuth = async () => { try { if (isLogin) { const result = await signInWithEmailAndPassword(auth, email, password); setUser(result.user); } else { const result = await createUserWithEmailAndPassword(auth, email, password); setUser(result.user); } } catch (err) { alert(err.message); } };

const placeBet = async () => { if (!betAmount || isNaN(betAmount)) return alert("Enter valid amount"); try { await addDoc(collection(db, "bets"), { userId: user.uid, email: user.email, amount: parseFloat(betAmount), status: "pending", placedAt: new Date(), }); setBetPlaced(true); } catch (err) { alert("Failed to place bet: " + err.message); } };

useEffect(() => { const handleKey = (e) => { if (e.ctrlKey && e.shiftKey && e.key === "A") { setIsAdmin(true); } }; window.addEventListener("keydown", handleKey); return () => window.removeEventListener("keydown", handleKey); }, []);

useEffect(() => { if (!gameStarted) return; const interval = setInterval(() => { setPosition((prev) => { const nextX = prev.x + 5; const nextY = prev.y + Math.sin((nextX / 50) * Math.PI) * 10; return { x: nextX, y: nextY }; }); setAngle(10); }, 50); return () => clearInterval(interval); }, [gameStarted]);

useEffect(() => { const q = query(collection(db, "bets"), orderBy("placedAt", "desc")); const unsub = onSnapshot(q, (snapshot) => { const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })); setScoreboard(data); }); return () => unsub(); }, []);

const forceCrash = async () => { setGameStarted(false); alert("Plane Crashed (Only you know this was manual)"); try { const q = query(collection(db, "bets"), where("status", "==", "pending")); const snapshot = await getDocs(q); snapshot.forEach(async (docSnap) => { const ref = doc(db, "bets", docSnap.id); await updateDoc(ref, { status: "lost" }); }); } catch (err) { console.error("Failed to update bets: ", err); } };

if (!user) { return ( <div className="w-full h-screen bg-gradient-to-b from-blue-100 to-blue-300 flex flex-col items-center justify-center gap-4"> <Card className="w-[300px] p-4 space-y-4"> <h2 className="text-xl font-bold text-center">{isLogin ? "Login" : "Sign Up"}</h2> <input type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} className="w-full p-2 border rounded" /> <input type="password" placeholder="Password" value={password} onChange={(e) => setPassword(e.target.value)} className="w-full p-2 border rounded" /> <Button onClick={handleAuth}>{isLogin ? "Login" : "Sign Up"}</Button> <p className="text-center text-sm cursor-pointer" onClick={() => setIsLogin(!isLogin)}> {isLogin ? "Don't have an account? Sign Up" : "Already have an account? Login"} </p> </Card> </div> ); }

return ( <div className="w-full h-screen bg-gradient-to-b from-blue-200 to-blue-400 flex flex-col items-center justify-center gap-4"> {!betPlaced && ( <Card className="w-[300px] p-4"> <h2 className="text-lg font-semibold mb-2">Place Your Bet</h2> <input type="number" placeholder="Enter amount" value={betAmount} onChange={(e) => setBetAmount(e.target.value)} className="w-full p-2 border rounded mb-2" /> <Button onClick={placeBet} className="w-full">Place Bet</Button> </Card> )}

{betPlaced && (
    <Card className="w-[400px] h-[400px] bg-white relative overflow-hidden">
      <CardContent className="w-full h-full">
        <motion.div
          className="w-12 h-12 rounded-full"
          animate={{ x: position.x, y: position.y, rotate: angle }}
          transition={{ type: "spring", stiffness: 100 }}
          style={{ backgroundImage: 'url("/plane.png")', backgroundSize: 'cover' }}
        />
      </CardContent>
    </Card>
  )}

  {betPlaced && (
    <Button onClick={() => setGameStarted(!gameStarted)}>
      {gameStarted ? "Stop Auto Fly" : "Start Auto Fly"}
    </Button>
  )}

  <Card className="w-[400px] max-h-[200px] overflow-y-auto p-2 bg-white border shadow-md">
    <h3 className="text-md font-semibold mb-2">Live Scoreboard</h3>
    {scoreboard.map((bet) => (
      <div key={bet.id} className="flex justify-between text-sm border-b py-1">
        <span>{bet.email}</span>
        <span>₹{bet.amount}</span>
        <span>{bet.status}</span>
      </div>
    ))}
  </Card>

  {isAdmin && (
    <div className="fixed top-4 right-4 bg-white border shadow-md p-4 rounded-xl space-y-2">
      <h2 className="text-lg font-bold mb-2">Admin Control</h2>
      <Button onClick={forceCrash} variant="destructive">Crash Now</Button>
      <Button onClick={() => setGameStarted(true)}>Start Again</Button>
    </div>
  )}
</div>

); }

