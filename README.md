# Lab Frida — Instrumentation Dynamique Mobile

> Installation · Déploiement · Analyse Android

---

## Objectifs

- Installer et vérifier Frida (client Python + CLI)
- Déployer et lancer `frida-server` sur un appareil Android
- Établir une connexion et injecter un script minimal
- Explorer la console interactive et observer les composants de l'app
- Hooker des fonctions natives (réseau, fichiers) et des méthodes Java (SharedPreferences, SQLite, Debug)

---

## Prérequis

- Python 3.8+ et `pip`
- `adb` (Android Platform Tools)
- Appareil Android 8+ avec Débogage USB activé
- Droits administrateur / sudo sur la machine

---

## Étape 1 — Installer le client Frida

```bash
pip install --upgrade frida frida-tools

# Vérification
frida --version
python -c "import frida; print(frida.__version__)"
```

---

## Étape 2 — Configurer ADB

```bash
adb version
adb devices     # doit afficher 'device', pas 'unauthorized'
```

---

## Étape 3 — Déployer frida-server sur Android

```bash
# 1. Identifier l'architecture
adb shell getprop ro.product.cpu.abi

# 2. Télécharger sur github.com/frida/frida/releases
#    ex : frida-server-17.x.x-android-x86_64.xz

# 3. Décompresser, pousser, lancer
unxz frida-server-*.xz
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server -l 0.0.0.0

# Redirection de ports
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

---

## Étape 4 — Tester la connexion

```bash
frida-ps -U       # liste les processus de l'appareil
frida-ps -Uai     # liste les apps installées
```

---

## Étape 5 — Première injection

**hello.js** — test API Java :
```javascript
Java.perform(function () {
  console.log("[+] Frida Java.perform OK");
});
```
```bash
frida -U -f com.example.app -l hello.js
# puis dans la console :
%resume
```

**hello_native.js** — test hook natif :
```javascript
console.log("[+] Script chargé");
Interceptor.attach(Module.getExportByName(null, "recv"), {
  onEnter(args) { console.log("[+] recv appelée"); }
});
```
```bash
frida -U -n "NomDuProcessus" -l hello_native.js
```

---

## Étape 6 — Console interactive Frida

```javascript
Process.arch                                                    // architecture
Process.mainModule                                              // module principal
Process.getModuleByName("libc.so")                             // infos libc
Process.getModuleByName("libc.so").getExportByName("recv")     // adresse recv
Process.enumerateModules()                                      // toutes les libs
Process.enumerateThreads()                                      // threads actifs
Process.enumerateRanges('r-x')                                 // zones mémoire exécutables
Java.available                                                  // runtime Java dispo ?
```

---

## Étape 7 — Observer chiffrement, réseau et fichiers

**Bibliothèques de chiffrement :**
```javascript
Process.enumerateModules().filter(m =>
  m.name.indexOf("ssl") !== -1 ||
  m.name.indexOf("crypto") !== -1 ||
  m.name.indexOf("boring") !== -1
)
```

**hook_network.js** :
```javascript
Interceptor.attach(Process.getModuleByName("libc.so").getExportByName("send"), {
  onEnter(args) { console.log("[send] len=" + args[2].toInt32()); }
});
Interceptor.attach(Process.getModuleByName("libc.so").getExportByName("recv"), {
  onEnter(args) { console.log("[recv] len=" + args[2].toInt32()); }
});
```
```bash
frida -U -n "app1" -l hook_network.js
```

**hook_file.js** :
```javascript
Interceptor.attach(Process.getModuleByName("libc.so").getExportByName("open"), {
  onEnter(args) { console.log("[open] " + args[0].readUtf8String()); }
});
```
```bash
frida -U -n "app1" -l hook_file.js
```

---

## Étape 8 — Hooking Java

**hook_prefs.js** — SharedPreferences :
```javascript
Java.perform(function () {
  var Impl = Java.use("android.app.SharedPreferencesImpl");
  Impl.getString.overload("java.lang.String", "java.lang.String").implementation = function (key, def) {
    var r = this.getString(key, def);
    console.log("[Prefs] " + key + " = " + r);
    return r;
  };
});
```

**hook_sqlite.js** — requêtes SQL :
```javascript
Java.perform(function () {
  var DB = Java.use("android.database.sqlite.SQLiteDatabase");
  DB.rawQuery.overload("java.lang.String", "[Ljava.lang.String;").implementation = function (sql, a) {
    console.log("[SQLite] " + sql);
    return this.rawQuery(sql, a);
  };
});
```

**hook_debug.js** — détection débogueur :
```javascript
Java.perform(function () {
  var D = Java.use("android.os.Debug");
  D.isDebuggerConnected.implementation = function () {
    var r = this.isDebuggerConnected();
    console.log("[Debug] isDebuggerConnected = " + r);
    return r;
  };
});
```

---

## Nettoyage

```bash
adb shell pkill -f frida-server
adb shell rm /data/local/tmp/frida-server
pip uninstall frida frida-tools
```

---

## Dépannage rapide

| Problème | Solution |
|---|---|
| `frida: command not found` | Vérifier le PATH Python — relancer `pip install frida frida-tools` |
| `unable to connect to remote frida-server` | Vérifier `adb devices` + `ps \| grep frida` + `adb forward` |
| Mismatch de versions | Aligner client et serveur sur la même version exacte |
| `Permission denied` | `chmod 755` sur frida-server — utiliser `/data/local/tmp` |
