# VRChat Crash Detection - Application de Sécurité

## 🎯 Vue d'ensemble

Cette application permet de détecter les clients et avatars crash pour VRChat, avec une intégration VRCX pour la sécurité de la plateforme.

**⚠️ IMPORTANT**: L'application actuelle utilise des données simulées pour la démonstration. Voici comment intégrer les vraies APIs.

## 🔧 État Actuel vs Intégration Réelle

### État Actuel (Démonstration)
- ✅ Interface utilisateur complète et fonctionnelle
- ✅ Système d'analyse avec données simulées
- ✅ Historique et paramètres avancés
- ✅ Compatible avec l'architecture VRCX
- ✅ API REST fonctionnelle avec mock data

### Intégration Réelle Nécessaire

#### 1. API VRChat Officielle

**Endpoint principal**: `https://api.vrchat.cloud/api/1/`

**Authentification requise**:
```javascript
// Dans votre .env.local
VRCHAT_USERNAME=votre_username
VRCHAT_PASSWORD=votre_password
VRCHAT_API_KEY=votre_api_key
```

**Endpoints utiles pour l'analyse**:
- `GET /users/{userId}` - Informations utilisateur
- `GET /avatars/{avatarId}` - Détails avatar
- `GET /users/{userId}/avatars` - Avatars d'un utilisateur
- `GET /worlds/{worldId}` - Informations monde

#### 2. Intégration VRCX

**VRCX Database Access**:
```javascript
// VRCX stocke ses données dans SQLite
// Chemin typique: %APPDATA%/VRCX/VRCX.sqlite3

// Tables importantes:
// - users: Informations utilisateurs
// - avatars: Données avatars
// - world_visits: Historique des mondes
// - friend_log: Journal d'amitié
```

## 🚀 Comment Implémenter l'Analyse Réelle

### Étape 1: Modifier l'API Backend

Remplacez le contenu de `src/app/api/detect/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'

// Configuration VRChat API
const VRCHAT_API_BASE = 'https://api.vrchat.cloud/api/1'
const VRCHAT_USER_AGENT = 'VRChatCrashDetection/1.0.0'

interface VRChatUser {
  id: string
  username: string
  displayName: string
  bio: string
  tags: string[]
  status: string
  statusDescription: string
  profilePicOverride: string
  userIcon: string
  last_login: string
  date_joined: string
  isFriend: boolean
  friendKey: string
}

interface VRChatAvatar {
  id: string
  name: string
  description: string
  authorId: string
  authorName: string
  tags: string[]
  assetUrl: string
  imageUrl: string
  thumbnailImageUrl: string
  releaseStatus: string
  version: number
  unityPackageUrl: string
}

// Fonction d'authentification VRChat
async function authenticateVRChat(): Promise<string> {
  const response = await fetch(`${VRCHAT_API_BASE}/auth/user`, {
    method: 'GET',
    headers: {
      'Authorization': `Basic ${Buffer.from(`${process.env.VRCHAT_USERNAME}:${process.env.VRCHAT_PASSWORD}`).toString('base64')}`,
      'User-Agent': VRCHAT_USER_AGENT
    }
  })
  
  if (!response.ok) {
    throw new Error('Échec de l\'authentification VRChat')
  }
  
  // Récupérer le cookie d'authentification
  const cookies = response.headers.get('set-cookie')
  const authCookie = cookies?.match(/auth=([^;]+)/)?.[1]
  
  if (!authCookie) {
    throw new Error('Cookie d\'authentification non trouvé')
  }
  
  return authCookie
}

// Analyse utilisateur réelle
async function analyzeRealUser(userId: string, authToken: string): Promise<any> {
  try {
    // Récupérer les données utilisateur
    const userResponse = await fetch(`${VRCHAT_API_BASE}/users/${userId}`, {
      headers: {
        'Cookie': `auth=${authToken}`,
        'User-Agent': VRCHAT_USER_AGENT
      }
    })
    
    if (!userResponse.ok) {
      throw new Error('Utilisateur non trouvé')
    }
    
    const userData: VRChatUser = await userResponse.json()
    
    // Analyse des indicateurs de crash client
    const suspiciousIndicators = analyzeSuspiciousPatterns(userData)
    
    return {
      type: suspiciousIndicators.length > 0 ? 'client crash' : 'clean',
      userName: userData.displayName || userData.username,
      profile: `https://vrchat.com/home/user/${userId}`,
      cause: suspiciousIndicators.join(', '),
      details: generateAnalysisDetails(userData, suspiciousIndicators),
      timestamp: new Date().toISOString(),
      analysisId: `analysis_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      isValid: true,
      realData: true // Indicateur que ce sont de vraies données
    }
  } catch (error) {
    throw new Error(`Erreur lors de l'analyse: ${error.message}`)
  }
}

// Analyse avatar réelle
async function analyzeRealAvatar(avatarId: string, authToken: string): Promise<any> {
  try {
    const avatarResponse = await fetch(`${VRCHAT_API_BASE}/avatars/${avatarId}`, {
      headers: {
        'Cookie': `auth=${authToken}`,
        'User-Agent': VRCHAT_USER_AGENT
      }
    })
    
    if (!avatarResponse.ok) {
      throw new Error('Avatar non trouvé')
    }
    
    const avatarData: VRChatAvatar = await avatarResponse.json()
    
    // Analyse des indicateurs d'avatar crash
    const crashIndicators = analyzeAvatarCrashPatterns(avatarData)
    
    return {
      type: crashIndicators.length > 0 ? 'avatar crash' : 'clean',
      userName: avatarData.authorName,
      profile: `https://vrchat.com/home/user/${avatarData.authorId}`,
      avatarName: avatarData.name,
      cause: crashIndicators.join(', '),
      details: generateAvatarAnalysisDetails(avatarData, crashIndicators),
      timestamp: new Date().toISOString(),
      analysisId: `analysis_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      isValid: true,
      realData: true
    }
  } catch (error) {
    throw new Error(`Erreur lors de l'analyse avatar: ${error.message}`)
  }
}

// Analyse des patterns suspects (utilisateur)
function analyzeSuspiciousPatterns(userData: VRChatUser): string[] {
  const indicators: string[] = []
  
  // Vérifications basées sur les données réelles
  if (userData.tags.includes('system_probable_troll')) {
    indicators.push('Marqué comme troll probable par le système')
  }
  
  if (userData.bio.toLowerCase().includes('crash') || 
      userData.bio.toLowerCase().includes('lag') ||
      userData.bio.toLowerCase().includes('freeze')) {
    indicators.push('Bio contient des références à des crashes')
  }
  
  if (userData.status === 'busy' && userData.statusDescription.includes('crash')) {
    indicators.push('Statut indique des activités de crash')
  }
  
  // Analyse du nom d'utilisateur
  const suspiciousNamePatterns = [
    /crash/i, /lag/i, /freeze/i, /ddos/i, /exploit/i
  ]
  
  if (suspiciousNamePatterns.some(pattern => pattern.test(userData.displayName))) {
    indicators.push('Nom d\'utilisateur suspect')
  }
  
  return indicators
}

// Analyse des patterns d'avatar crash
function analyzeAvatarCrashPatterns(avatarData: VRChatAvatar): string[] {
  const indicators: string[] = []
  
  // Vérifications basées sur les métadonnées
  if (avatarData.name.toLowerCase().includes('crash') ||
      avatarData.name.toLowerCase().includes('lag') ||
      avatarData.name.toLowerCase().includes('freeze')) {
    indicators.push('Nom d\'avatar suspect')
  }
  
  if (avatarData.description.toLowerCase().includes('crash') ||
      avatarData.description.toLowerCase().includes('performance')) {
    indicators.push('Description contient des références à des problèmes')
  }
  
  // Vérification des tags
  const suspiciousTags = avatarData.tags.filter(tag => 
    tag.includes('crash') || 
    tag.includes('lag') || 
    tag.includes('performance')
  )
  
  if (suspiciousTags.length > 0) {
    indicators.push(`Tags suspects: ${suspiciousTags.join(', ')}`)
  }
  
  return indicators
}

// Génération des détails d'analyse
function generateAnalysisDetails(userData: VRChatUser, indicators: string[]): string {
  if (indicators.length === 0) {
    return 'Utilisateur vérifié - Aucune activité suspecte détectée dans les données VRChat'
  }
  
  return `Analyse VRChat révèle ${indicators.length} indicateur(s) suspect(s). ` +
         `Dernière connexion: ${userData.last_login}. ` +
         `Membre depuis: ${userData.date_joined}. ` +
         `Recommandation: Surveillance accrue recommandée.`
}

function generateAvatarAnalysisDetails(avatarData: VRChatAvatar, indicators: string[]): string {
  if (indicators.length === 0) {
    return 'Avatar vérifié - Conforme aux standards VRChat selon les métadonnées'
  }
  
  return `Avatar analysé via API VRChat. ${indicators.length} problème(s) détecté(s). ` +
         `Version: ${avatarData.version}. ` +
         `Statut: ${avatarData.releaseStatus}. ` +
         `Recommandation: Éviter l'utilisation de cet avatar.`
}

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    
    if (!body.id || typeof body.id !== 'string') {
      return NextResponse.json(
        { error: 'ID manquant ou invalide' },
        { status: 400 }
      )
    }

    const id = body.id.trim()
    
    // Authentification VRChat
    const authToken = await authenticateVRChat()
    
    if (id.startsWith('usr_')) {
      const result = await analyzeRealUser(id, authToken)
      return NextResponse.json(result)
    } else if (id.startsWith('avtr_')) {
      const result = await analyzeRealAvatar(id, authToken)
      return NextResponse.json(result)
    } else {
      return NextResponse.json(
        { error: 'Format d\'ID invalide' },
        { status: 400 }
      )
    }
  } catch (error) {
    console.error('Erreur API:', error)
    return NextResponse.json(
      { error: error.message },
      { status: 500 }
    )
  }
}
```

### Étape 2: Intégration VRCX

Créez `src/lib/vrcx-integration.ts`:

```typescript
import Database from 'better-sqlite3'
import path from 'path'
import os from 'os'

// Chemin vers la base de données VRCX
const VRCX_DB_PATH = path.join(os.homedir(), 'AppData', 'Roaming', 'VRCX', 'VRCX.sqlite3')

export class VRCXIntegration {
  private db: Database.Database

  constructor() {
    try {
      this.db = new Database(VRCX_DB_PATH, { readonly: true })
    } catch (error) {
      throw new Error('VRCX database non trouvée. Assurez-vous que VRCX est installé.')
    }
  }

  // Récupérer les données utilisateur depuis VRCX
  getUserData(userId: string) {
    const stmt = this.db.prepare(`
      SELECT * FROM users 
      WHERE id = ? 
      ORDER BY updated_at DESC 
      LIMIT 1
    `)
    return stmt.get(userId)
  }

  // Récupérer l'historique des mondes visités
  getWorldHistory(userId: string) {
    const stmt = this.db.prepare(`
      SELECT w.*, wv.created_at as visit_time
      FROM world_visits wv
      JOIN worlds w ON w.id = wv.world_id
      WHERE wv.user_id = ?
      ORDER BY wv.created_at DESC
      LIMIT 50
    `)
    return stmt.all(userId)
  }

  // Analyser les patterns suspects dans VRCX
  analyzeSuspiciousActivity(userId: string) {
    const userData = this.getUserData(userId)
    const worldHistory = this.getWorldHistory(userId)
    
    const suspiciousIndicators = []
    
    // Vérifier les mondes suspects
    const suspiciousWorlds = worldHistory.filter(world => 
      world.name?.toLowerCase().includes('crash') ||
      world.name?.toLowerCase().includes('lag') ||
      world.description?.toLowerCase().includes('crash')
    )
    
    if (suspiciousWorlds.length > 0) {
      suspiciousIndicators.push(`Visite de ${suspiciousWorlds.length} monde(s) suspect(s)`)
    }
    
    // Vérifier la fréquence des changements de monde (indicateur de crash)
    const recentVisits = worldHistory.filter(visit => 
      new Date(visit.visit_time) > new Date(Date.now() - 24 * 60 * 60 * 1000)
    )
    
    if (recentVisits.length > 50) {
      suspiciousIndicators.push('Changements de monde excessifs (possible crash client)')
    }
    
    return {
      userData,
      worldHistory,
      suspiciousIndicators,
      riskLevel: suspiciousIndicators.length > 2 ? 'high' : 
                 suspiciousIndicators.length > 0 ? 'medium' : 'low'
    }
  }

  // Exporter les données vers VRCX
  exportAnalysisToVRCX(analysisData: any) {
    // Insérer les résultats d'analyse dans une table personnalisée
    const stmt = this.db.prepare(`
      INSERT OR REPLACE INTO crash_analysis 
      (user_id, analysis_result, risk_level, timestamp, details)
      VALUES (?, ?, ?, ?, ?)
    `)
    
    return stmt.run(
      analysisData.userId,
      analysisData.type,
      analysisData.riskLevel,
      new Date().toISOString(),
      JSON.stringify(analysisData)
    )
  }

  close() {
    this.db.close()
  }
}
```

### Étape 3: Configuration Environnement

Créez `.env.local`:

```env
# VRChat API Credentials
VRCHAT_USERNAME=votre_username_vrchat
VRCHAT_PASSWORD=votre_mot_de_passe_vrchat
VRCHAT_API_KEY=votre_api_key

# VRCX Integration
VRCX_DB_PATH=C:/Users/VotreNom/AppData/Roaming/VRCX/VRCX.sqlite3
ENABLE_VRCX_INTEGRATION=true

# Security
ANALYSIS_API_KEY=votre_cle_api_securisee
ENABLE_REAL_TIME_ANALYSIS=false
```

## 🔒 Considérations de Sécurité

### Limitations VRChat API
- **Rate Limiting**: 100 requêtes/minute
- **Authentification**: Requiert compte VRChat valide
- **Permissions**: Accès limité aux données publiques
- **ToS Compliance**: Respecter les conditions d'utilisation

### Détection Réelle vs Simulée

**Ce que l'API VRChat peut détecter**:
- ✅ Informations de profil utilisateur
- ✅ Métadonnées d'avatar
- ✅ Statut et bio utilisateur
- ✅ Tags système (troll probable, etc.)
- ✅ Historique public limité

**Ce que l'API VRChat NE peut PAS détecter**:
- ❌ Modifications client en temps réel
- ❌ Code malveillant dans les avatars
- ❌ Activité de crash en cours
- ❌ Données privées utilisateur
- ❌ Analyse technique approfondie

### Recommandations

1. **Utilisez les données VRChat comme indicateurs**, pas comme preuve absolue
2. **Combinez avec l'analyse VRCX** pour plus de contexte
3. **Implémentez un système de scoring** plutôt que des détections binaires
4. **Respectez la vie privée** et les ToS de VRChat
5. **Utilisez des seuils de confiance** pour éviter les faux positifs

## 🚀 Démarrage Rapide

1. **Installer les dépendances**:
```bash
npm install better-sqlite3 @types/better-sqlite3
```

2. **Configurer les variables d'environnement** dans `.env.local`

3. **Tester l'intégration**:
```bash
npm run dev
```

4. **Vérifier la connexion VRCX**:
   - VRCX doit être installé et avoir des données
   - Le chemin de la base de données doit être correct

## 📊 Métriques et Monitoring

L'application peut tracker:
- Nombre d'analyses effectuées
- Taux de détection de menaces
- Performance des requêtes API
- Intégration VRCX status
- Erreurs et exceptions

## 🤝 Contribution

Pour contribuer au développement:
1. Fork le repository
2. Créer une branche feature
3. Tester avec de vraies données VRChat
4. Soumettre une pull request

## ⚠️ Avertissements Légaux

- Cette application est destinée à des fins de sécurité communautaire
- Respectez les ToS de VRChat et VRCX
- N'utilisez pas pour harceler ou cibler des utilisateurs
- Les résultats sont indicatifs, pas définitifs
- Utilisez de manière responsable et éthique

---

**Status**: ✅ Interface complète | ⚠️ APIs réelles à implémenter | 🔄 Intégration VRCX en cours
