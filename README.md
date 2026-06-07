# LAB-17-Cracker-OWASP-Uncrackable-Android-Level-3-Cours-S-curit-des-applications-mobiles
# LAB-17-Cracker-OWASP-Uncrackable-Android-Level-3-Cours-S-curit-des-applications-mobiles
# 🔐 Lab Sécurité Android — Reverse Engineering & Bypass de Protections

> **Niveau :** Intermédiaire / Avancé  
> **Durée estimée :** 4 à 6 heures  
> **Outils :** 100% gratuits

---

## 📋 Description

Ce codelab te guide pas à pas dans l'analyse et le contournement des mécanismes de protection d'une application Android. Tu vas travailler sur une APK volontairement vulnérable pour apprendre les techniques de reverse engineering utilisées en pentest mobile.

Tout le lab est réalisable avec des outils open source ou gratuits — **aucune licence payante n'est requise**.

---

## 🎯 Objectifs d'apprentissage

À la fin de ce codelab, tu sauras :

- ✅ **Décompiler et patcher une APK** (smali + Java)
- ✅ **Analyser une librairie native** (`.so`) avec un outil gratuit
- ✅ **Contourner les protections anti-debug, anti-Frida, anti-root** et vérification d'intégrité
- ✅ **Déboguer en live avec GDB** et modifier des registres à la volée
- ✅ **Comprendre et calculer un XOR byte par byte** pour retrouver un mot de passe secret
- ✅ Maîtriser un **workflow de pentest Android complet** avec uniquement des outils gratuits

---

## 🛠️ Outils utilisés

| Outil | Rôle | Lien |
|---|---|---|
| **Android Studio** | Émulateur Android + ADB | [developer.android.com](https://developer.android.com/studio) |
| **apktool** | Décompilation / recompilation APK | [apktool.org](https://apktool.org) |
| **Ghidra** | Analyse de librairies natives `.so` | [ghidra-sre.org](https://ghidra-sre.org) |
| **GDB + gdbserver** | Débogage dynamique natif | Inclus dans NDK Android |
| **jadx** *(optionnel)* | Décompilation Java lisible | [github.com/skylot/jadx](https://github.com/skylot/jadx) |

---

## 📁 Structure du projet

```
android-security-lab/
│
├── README.md                  ← Ce fichier
├── target/
│   └── vulnerable-app.apk     ← APK cible du lab
│
├── patched/
│   └── vulnerable-app-patched.apk  ← APK modifiée après patch smali
│
├── analysis/
│   ├── smali/                 ← Code smali décompilé
│   ├── native/                ← Fichiers .so extraits
│   └── ghidra-project/        ← Projet Ghidra pour l'analyse statique
│
└── writeup/
    └── solution.md            ← Notes et writeup de la solution
```

---

## 🚀 Mise en place de l'environnement

### 1. Prérequis système

- OS : Windows 10/11, macOS ou Linux (Ubuntu recommandé)
- RAM : minimum 8 Go (16 Go recommandés pour l'émulateur)
- Espace disque : ~10 Go

### 2. Installation des outils

```bash
# apktool
wget https://bitbucket.org/iBotPeaches/apktool/downloads/apktool_2.9.3.jar -O apktool.jar
# Wrapper Linux/macOS : https://apktool.org/docs/install

# jadx (optionnel mais recommandé)
wget https://github.com/skylot/jadx/releases/latest/download/jadx-1.5.0.zip
unzip jadx-1.5.0.zip -d jadx/

# Ghidra : télécharger depuis https://ghidra-sre.org
# Android Studio : https://developer.android.com/studio
```

### 3. Configuration de l'émulateur

Dans Android Studio, créer un AVD avec :
- **API Level** : 29 (Android 10) ou inférieur pour faciliter le débogage natif
- **Architecture** : x86 (plus rapide) ou x86_64
- **RAM** : 2048 Mo minimum

```bash
# Vérifier la connexion ADB
adb devices

# Installer l'APK cible
adb install target/vulnerable-app.apk
```

---

## 📖 Déroulé du Lab

### Module 1 — Décompilation de l'APK

```bash
# Décompiler l'APK avec apktool
java -jar apktool.jar d target/vulnerable-app.apk -o analysis/smali/

# Ouvrir le code Java lisible avec jadx
jadx/bin/jadx-gui target/vulnerable-app.apk
```

🎯 **Objectif :** Identifier les points d'entrée de l'application et les appels vers la librairie native.

---

### Module 2 — Contournement des protections (Anti-Debug / Anti-Root / Anti-Frida)

Les protections courantes se trouvent généralement dans des méthodes appelées au démarrage de l'app. Dans le code smali, repère des appels comme :

```smali
invoke-virtual {v0}, Lcom/example/app/SecurityCheck;->isRooted()Z
invoke-virtual {v0}, Lcom/example/app/SecurityCheck;->isFridaDetected()Z
```

**Patch smali :** Remplacer le résultat par `false` pour court-circuiter la vérification.

```smali
# AVANT
invoke-virtual {v0}, Lcom/example/app/SecurityCheck;->isRooted()Z
move-result v1

# APRÈS (bypass direct)
const/4 v1, 0x0   # Force la valeur à false
```

Après modification, recompiler et signer l'APK :

```bash
java -jar apktool.jar b analysis/smali/ -o patched/vulnerable-app-patched.apk
# Signer avec uber-apk-signer ou keytool + apksigner
```

---

### Module 3 — Analyse de la librairie native avec Ghidra

```bash
# Extraire le .so de l'APK
unzip target/vulnerable-app.apk lib/x86/libnative.so -d analysis/native/
```

Dans Ghidra :
1. Créer un nouveau projet → Importer `libnative.so`
2. Lancer l'auto-analyse
3. Chercher la fonction de validation du mot de passe (souvent `Java_com_example_*_checkPassword`)

🎯 **Objectif :** Identifier l'algorithme de validation et trouver la clé XOR utilisée.

---

### Module 4 — Calcul du mot de passe par XOR

Une fois la clé XOR identifiée dans Ghidra, calcule le mot de passe en Python :

```python
# Exemple de calcul XOR byte par byte
cipher = [0x41, 0x1F, 0x3C, 0x2A, 0x55]  # Bytes extraits de Ghidra
key    = [0x1A, 0x7B, 0x4E, 0x5F, 0x30]  # Clé XOR trouvée

password = ''.join(chr(c ^ k) for c, k in zip(cipher, key))
print(f"Mot de passe : {password}")
```

---

### Module 5 — Débogage dynamique avec GDB

```bash
# Pousser gdbserver sur l'émulateur
adb push <ndk-path>/prebuilt/android-x86/gdbserver/gdbserver /data/local/tmp/
adb shell chmod +x /data/local/tmp/gdbserver

# Attacher gdbserver au processus de l'app
adb shell /data/local/tmp/gdbserver :5039 --attach $(adb shell pidof com.example.app)

# Rediriger le port et lancer GDB côté hôte
adb forward tcp:5039 tcp:5039
gdb
(gdb) target remote :5039
(gdb) info registers
(gdb) set $eax = 1     # Modifier un registre pour forcer un résultat
(gdb) continue
```

🎯 **Objectif :** Modifier le registre de retour d'une fonction de vérification pour la court-circuiter en live.

---

## ✅ Critères de réussite

| Étape | Validé quand... |
|---|---|
| Module 1 | L'APK est décompilée sans erreur, le code smali est lisible |
| Module 2 | L'app se lance sans détection de root/Frida après patch |
| Module 3 | La fonction de validation est identifiée dans Ghidra |
| Module 4 | Le mot de passe secret est calculé et accepté par l'app |
| Module 5 | Un registre est modifié en live via GDB avec succès |

---

## 📚 Ressources complémentaires

- [OWASP Mobile Security Testing Guide (MASTG)](https://mas.owasp.org/MASTG/)
- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [Android Smali cheatsheet](https://github.com/JesusFreke/smali)
- [Ghidra documentation](https://ghidra-sre.org/documentation.html)
- [apktool documentation](https://apktool.org/docs/the-basics)

---

## ⚠️ Avertissement légal

> Ce lab est **strictement à des fins éducatives**. Les techniques présentées ne doivent être appliquées que sur des applications et systèmes pour lesquels tu disposes d'une **autorisation explicite**. Toute utilisation malveillante est illégale et contraire à l'éthique professionnelle.

---

## 👤 Auteur

**Hiba** — Étudiante en cybersécurité & développement logiciel  
*Codelab réalisé dans le cadre d'un cours de sécurité mobile*

---

*Dernière mise à jour : Juin 2026*
