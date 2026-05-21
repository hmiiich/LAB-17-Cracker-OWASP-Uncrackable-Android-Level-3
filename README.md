OWASP MSTG UnCrackable Level 3 – Writeup
Overview

Ce rapport présente l’analyse complète de l’application Android OWASP MSTG UnCrackable Level 3.
L’objectif du challenge est de :

contourner les protections anti-root / anti-debug / anti-tamper ;
analyser le code Java et natif ;
patcher l’APK ;
récupérer la chaîne secrète cachée dans la librairie native ;
valider le secret dans l’application.
Tools Used
ADB
JADX
Apktool
Android Studio / IntelliJ IDEA
keytool
apksigner
Ghidra
Python
1. APK Installation

L’APK a été installé avec succès sur un émulateur Android via ADB.

adb devices
adb install UnCrackable-Level3.apk

L’émulateur détecté :

emulator-5554    device
2. First Launch

Au lancement de l’application, un message de protection apparaît immédiatement :

Rooting or tampering detected.
This is unacceptable. The app is now going to exit.

Cela confirme la présence de mécanismes de :

détection de root ;
anti-debug ;
détection de modification (tamper detection).
3. Static Analysis with JADX

L’APK a été ouvert avec JADX afin d’inspecter le code Java.

Le package principal identifié :

sg.vantagepoint.uncrackable3

Classes importantes observées :

MainActivity
CodeCheck
BuildConfig
RootDetection
IntegrityCheck
4. Integrity Check Analysis

Dans MainActivity, la méthode verifyLibs() vérifie l’intégrité des composants critiques de l’application.

L’application calcule les CRC de :

classes.dex
librairies natives (libfoo.so)

Puis compare ces valeurs avec celles stockées dans les ressources internes.

Exemple logique observée :

if (calculated_crc != expected_crc) {
    tampered = true;
}

Si une différence est détectée, l’application considère qu’elle a été modifiée.

5. Root, Debug and Tamper Detection

Dans onCreate(), plusieurs contrôles de sécurité sont exécutés :

RootDetection.checkRoot1();
RootDetection.checkRoot2();
RootDetection.checkRoot3();
IntegrityCheck.isDebuggable(getApplicationContext());

Si une condition retourne true, l’application déclenche :

showDialog("Rooting or tampering detected.");

Les protections incluent :

détection de root ;
vérification des binaires su;
détection de debugger ;
vérification d’intégrité.
6. Secret Verification Function

La méthode responsable de la validation du secret est :

verify(View view)

Cette fonction récupère le texte saisi puis appelle :

this.check.check_code(string)

Si le secret est correct, l’application affiche :

Success!
This is the correct secret.
7. APK Decompilation with Apktool

Afin de patcher les protections, l’APK a été décompilé avec Apktool.

apktool d UnCrackable-Level3.apk -o uncrackable3

La structure du projet obtenue :

lib/
res/
smali/
AndroidManifest.xml
apktool.yml
8. Smali Code Analysis

Le fichier principal analysé :

smali/sg/vantagepoint/uncrackable3/MainActivity.smali

Le dialogue de détection est invoqué via :

const-string v0, "Rooting or tampering detected."

invoke-direct {p0, v0},
Lsg/vantagepoint/uncrackable3/MainActivity;->showDialog(Ljava/lang/String;)V

Les contrôles root/debug/tamper sont exécutés avant l’initialisation normale de l’application.

9. Patch the Application

Le code Smali a été modifié afin de :

bypasser les contrôles de sécurité ;
empêcher l’affichage du message d’erreur ;
permettre l’exécution normale de l’application.

Le bloc conditionnel responsable du showDialog() a été neutralisé.

Exemple de logique patchée :

if-eqz v0, :continue_execution

Ou suppression directe du saut conditionnel.

10. Rebuild the APK

Après modification des fichiers Smali, l’APK a été recompilé :

apktool b uncrackable3 -o UnCrackable-Level3-patched.apk
11. Generate a Keystore

Comme l’APK recompilé n’est plus signé, une nouvelle clé a été générée :

keytool -genkey -v \
-keystore my-release-key.jks \
-keyalg RSA \
-keysize 2048 \
-validity 10000 \
-alias my-alias
12. Sign the APK

Signature de l’APK avec apksigner :

apksigner sign \
--ks my-release-key.jks \
--out UnCrackable-Level3-final.apk \
UnCrackable-Level3-patched.apk
13. Install the Final APK

Installation de l’APK patché :

adb install UnCrackable-Level3-final.apk

Après installation, l’application démarre correctement sans afficher l’alerte de sécurité.

14. Native Library Analysis with Ghidra

L’application charge une librairie native :

System.loadLibrary("foo");

Le fichier libfoo.so a été ouvert dans Ghidra afin d’analyser le code natif.

Les éléments observés :

protections anti-Frida ;
chaînes encodées ;
logique de validation du secret ;
opérations XOR.

Une fonction native récupère une séquence de bytes encodés puis applique un XOR dynamique.

15. Decode the Secret with Python

Après reverse engineering du code natif, une séquence encodée ainsi qu’une clé XOR ont été récupérées.

Script Python utilisé :

# === XOR SECRET DECODER ===

encoded = bytes.fromhex(
    "1d0811130f1749150d0003195a1d1315080e5a0017081314"
)

xor_key = b"pizzapizzapizzapizzapizza"

secret = bytes(a ^ b for a, b in zip(encoded, xor_key))

print("Recovered secret:", secret.decode())

Exécution :

python decode.py

Résultat :

Recovered secret: making owasp great again
16. Validate the Secret

Le secret récupéré a été saisi dans l’application :

making owasp great again

L’application affiche alors :

Success!
This is the correct secret.
Final Secret
making owasp great again
Conclusion

Ce challenge a permis de pratiquer plusieurs techniques avancées de reverse engineering Android :

analyse statique Java ;
modification Smali ;
bypass anti-root ;
bypass anti-debug ;
recompilation et signature APK ;
reverse engineering natif avec Ghidra ;
déchiffrement XOR ;
validation dynamique du secret.

Le challenge démontre également l’importance des protections natives dans les applications Android modernes, ainsi que leurs limites face à une analyse approfondie.

<img width="1601" height="301" alt="Screenshot 2026-05-21 191035" src="https://github.com/user-attachments/assets/e325ba6c-221d-47dd-9c52-3983e545be25" />

<img width="1417" height="745" alt="IMG2" src="https://github.com/user-attachments/assets/efa57bca-f01b-4d2d-83c2-93e64ad80cbb" />
<img width="1356" height="712" alt="IMG3" src="https://github.com/user-attachments/assets/08b573dc-65ef-48d8-9304-e8e67b8812c0" />

<img width="1093" height="171" alt="IMG4" src="https://github.com/user-attachments/assets/4ff80924-c981-4f7b-ba6d-f9d69f964263" />

<img width="1090" height="953" alt="IMG5" src="https://github.com/user-attachments/assets/8f4d9667-e6e0-4201-a2ea-4674cdc93763" />

<img width="1475" height="1156" alt="IMG6" src="https://github.com/user-attachments/assets/47d31bfa-c9d5-42ab-91ad-0d9eb0eac85d" />
<img width="1438" height="1170" alt="IMG7" src="https://github.com/user-attachments/assets/a35e1041-adcf-40c5-862b-c62e9444f06e" />

<img width="1452" height="353" alt="IMG8" src="https://github.com/user-attachments/assets/c1cc8fab-3f23-43ea-9878-770059389764" />
<img width="362" height="117" alt="IMG9" src="https://github.com/user-attachments/assets/f961196c-b323-4213-9e48-a459a64c1058" />

<img width="1429" height="663" alt="IMG10" src="https://github.com/user-attachments/assets/c2461029-4484-4eee-b3c1-0cbadd888dc4" />

<img width="521" height="161" alt="IMG11" src="https://github.com/user-attachments/assets/975d234d-d80e-458c-9f7f-e7878d09379a" />















