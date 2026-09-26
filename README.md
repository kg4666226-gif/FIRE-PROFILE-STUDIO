export interface ProfileData {
‎  uid: string;
‎  name: string;
‎  nickname: string;
‎  level: number;
‎  rank: string;
‎  title: string;
‎  region: string;
‎  badge: string;
‎  likes: number;
‎  xp: number;
‎  xp_max: number;
‎  character: string;
‎  avatar_url: string;
‎  role: string;
‎  sound: boolean;
‎  status: string;
‎  coins: number;
‎  diamonds: number;
‎  weapons: string[];
‎  bannerTheme: string;
‎  kdRatio: number;
‎  headshotRate: number;
‎  cardTheme: 'red-fire' | 'cyber-blue' | 'toxic-green' | 'royal-gold' | 'neon-purple';
‎  sensitivity?: SensitivityConfig;
‎  lastLoginTimestamp?: number;
‎  loginStreak?: number;
‎}
‎
‎export interface SensitivityConfig {
‎  general: number;
‎  redDot: number;
‎  scope2x: number;
‎  scope4x: number;
‎  sniperScope: number;
‎  freeLook: number;
‎}
‎
‎export interface Mission {
‎  id: string;
‎  title: string;
‎  desc: string;
‎  progress: number;
‎  target: number;
‎  reward_coins: number;
‎  reward_xp: number;
‎  completed: boolean;
‎  claimed: boolean;
‎}import { initializeApp, getApps } from 'firebase/app';
‎import { 
‎  getFirestore, 
‎  doc, 
‎  getDoc, 
‎  setDoc, 
‎  collection, 
‎  getDocs, 
‎  writeBatch 
‎} from 'firebase/firestore';
‎import type { ProfileData, Mission } from '../src/types';
‎import fs from 'fs';
‎import path from 'path';
‎
‎let app;
‎let db: any = null;
‎
‎try {
‎  const configPath = path.resolve(process.cwd(), 'firebase-applet-config.json');
‎  if (fs.existsSync(configPath)) {
‎    const configData = JSON.parse(fs.readFileSync(configPath, 'utf8'));
‎    const firebaseConfig = {
‎      apiKey: configData.apiKey || 'placeholder-api-key',
‎      authDomain: configData.authDomain,
‎      projectId: configData.projectId,
‎      storageBucket: configData.storageBucket,
‎      messagingSenderId: configData.messagingSenderId,
‎      appId: configData.appId,
‎      firestoreDatabaseId: configData.firestoreDatabaseId || '(default)'
‎    };
‎    app = getApps().length === 0 ? initializeApp(firebaseConfig) : getApps()[0];
‎    db = getFirestore(app, firebaseConfig.firestoreDatabaseId);
‎    console.log('[Firestore] Connected to Cloud Firestore:', firebaseConfig.projectId);
‎  }
‎} catch (err) {
‎  console.warn('[Firestore] Initialization error, falling back to local memory store', err);
‎}
‎
‎export async function getProfileFromDB(uid: string): Promise<ProfileData | null> {
‎  if (!db) return null;
‎  const docRef = doc(db, 'profiles', uid);
‎  const snap = await getDoc(docRef);
‎  if (snap.exists()) {
‎    return snap.data() as ProfileData;
‎  }
‎  return null;
‎}
‎
‎export async function saveProfileToDB(profile: ProfileData): Promise<void> {
‎  if (!db) return;
‎  const docRef = doc(db, 'profiles', profile.uid);
‎  await setDoc(docRef, {
‎    ...profile,
‎    updatedAt: new Date().toISOString()
‎  }, { merge: true });
‎}
‎
‎export async function getMissionsFromDB(): Promise<Mission[]> {
‎  if (!db) return [];
‎  const colRef = collection(db, 'missions');
‎  const snap = await getDocs(colRef);
‎  const list: Mission[] = [];
‎  snap.forEach(docSnap => {
‎    list.push({ id: docSnap.id, ...docSnap.data() } as Mission);
‎  });
‎  return list;
‎}import { Router } from 'express';
‎import { getProfileFromDB, saveProfileToDB } from '../db';
‎import { validateProfileUpdate } from '../validators';
‎import { DEFAULT_PROFILE } from '../../src/data/constants';
‎
‎const router = Router();
‎
‎router.get('/:uid', async (req, res) => {
‎  try {
‎    const { uid } = req.params;
‎    let profile = await getProfileFromDB(uid);
‎    if (!profile) {
‎      profile = { ...DEFAULT_PROFILE, uid };
‎      await saveProfileToDB(profile);
‎    }
‎    res.json({ success: true, data: profile });
‎  } catch (err: any) {
‎    res.status(500).json({ success: false, error: err.message });
‎  }
‎});
‎
‎router.put('/:uid', async (req, res) => {
‎  try {
‎    const { uid } = req.params;
‎    const validatedData = validateProfileUpdate(req.body);
‎    let currentProfile = await getProfileFromDB(uid) || { ...DEFAULT_PROFILE, uid };
‎    
‎    const updatedProfile = {
‎      ...currentProfile,
‎      ...validatedData,
‎      uid
‎    };
‎
‎    await saveProfileToDB(updatedProfile);
‎    res.json({ success: true, data: updatedProfile });
‎  } catch (err: any) {
‎    res.status(400).json({ success: false, error: err.message });
‎  }
‎});
‎
‎export default router;
‎import React, { useState } from 'react';
‎import { Flame, Coins, Gem, Gift, Check, Clock, Zap, RotateCcw } from 'lucide-react';
‎import confetti from 'canvas-confetti';
‎import type { ProfileData } from '../types';
‎import { soundFX } from '../utils/audio';
‎
‎const DAILY_STREAK_REWARDS = [
‎  { day: 1, coins: 500, diamonds: 0 },
‎  { day: 2, coins: 750, diamonds: 0 },
‎  { day: 3, coins: 1000, diamonds: 15 },
‎  { day: 4, coins: 1250, diamonds: 0 },
‎  { day: 5, coins: 1500, diamonds: 30 },
‎  { day: 6, coins: 2000, diamonds: 50 },
‎  { day: 7, coins: 3500, diamonds: 120 },
‎];
‎
‎export const HomeTab: React.FC<{
‎  data: ProfileData;
‎  onUpdateProfile: (updates: Partial<ProfileData>, msg?: string, type?: any) => void;
‎}> = ({ data, onUpdateProfile }) => {
‎  const now = new Date();
‎  const lastLogin = data.lastLoginTimestamp ? new Date(data.lastLoginTimestamp) : null;
‎  const isClaimedToday = lastLogin ? lastLogin.toDateString() === now.toDateString() : false;
‎  const currentStreak = data.loginStreak || 1;
‎  const activeDay = ((currentStreak - 1) % 7) + 1;
‎  const reward = DAILY_STREAK_REWARDS[activeDay - 1];
‎
‎  const handleClaim = () => {
‎    if (isClaimedToday) return;
‎    soundFX.playReward(data.sound);
‎    confetti({ particleCount: 75, spread: 60 });
‎
‎    onUpdateProfile({
‎      coins: (data.coins || 0) + reward.coins,
‎      diamonds: (data.diamonds || 0) + reward.diamonds,
‎      loginStreak: currentStreak + 1,
‎      lastLoginTimestamp: Date.now()
‎    }, `CLAIMED DAY ${activeDay} REWARD: +${reward.coins} COINS`, 'gold');
‎  };
‎
‎  return (
‎    <div className="space-y-6">
‎      <div className="bg-[#121212] border-2 border-amber-500/40 rounded-xl p-5 shadow-xl">
‎        <div className="flex justify-between items-center mb-4">
‎          <div className="flex items-center gap-2">
‎            <Flame className="w-5 h-5 text-amber-400" />
‎            <h3 className="text-lg font-black text-amber-400">DAILY LOGIN STREAK</h3>
‎          </div>
‎          <span className="text-amber-400 font-mono font-bold">{currentStreak} DAYS</span>
‎        </div>
‎
‎        <button
‎          onClick={handleClaim}
‎          disabled={isClaimedToday}
‎          className={`w-full py-3 rounded-lg font-black uppercase font-mono ${
‎            isClaimedToday 
‎              ? 'bg-emerald-950 text-emerald-400 border border-emerald-500/40' 
‎              : 'bg-amber-500 hover:bg-amber-400 text-black shadow-lg cursor-pointer'
‎          }`}
‎        >
‎          {isClaimedToday ? '✓ CLAIMED FOR TODAY' : `CLAIM DAY ${activeDay} (+${reward.coins} COINS)`}
‎        </button>
‎      </div>
‎    </div>
‎  );
‎};{
‎  "appId": "com.fireprofilestudio.app",
‎  "appName": "Fire Profile Studio",
‎  "webDir": "dist",
‎  "bundledWebRuntime": false,
‎  "server": {
‎    "androidScheme": "https",
‎    "cleartext": true
‎  }
‎}ame: Build Android APK
‎on: [push, workflow_dispatch]
‎
‎jobs:
‎  build:
‎    runs-on: ubuntu-latest
‎    steps:
· ‎uses: actions/checkout@v4
· ‎uses: actions/setup-node@v4
‎        with:
‎          node-version: 20
· ‎uses: actions/setup-java@v4
‎        with:
‎          distribution: 'zulu'
‎          java-version: '17'
· ‎uses: android-actions/setup-android@v3
· ‎run: npm install && npm install @capacitor/core @capacitor/cli @capacitor/android
· ‎run: npm run build
· ‎run: npx cap add android || true && npx cap sync android
· ‎run: cd android && chmod +x ./gradlew && ./gradlew assembleDebug --no-daemon
· ‎uses: actions/upload-artifact@v4
‎        with:
‎          name: FireProfileStudio-debug.apk
‎          path: android/app/build/outputs/apk/debug/app-debug.apk
‎rules_version = '2';
‎service cloud.firestore {
‎  match /databases/{database}/documents {
‎    match /profiles/{uid} {
‎      allow read: if true;
‎      allow create, update: if true;
‎      allow delete: if false;
‎    }
‎    match /missions/{missionId} {
‎      allow read: if true;
‎      allow create, update: if true;
‎      allow delete: if false;
‎    }
‎  }
‎}# FIRE-PROFILE-STUDIO
Gay ek image editor hai

