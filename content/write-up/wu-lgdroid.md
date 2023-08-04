# LGDroid 

- [X] écriture
- [X] abstract
- [X] publication
- [~] post linkedin

📱🔍 [Quelles données dans un smartphone ?] 🕵️‍♂️

Dans ce write-up issu du challenge "LGDroid" de la plateforme Cyberdefenders, j'analyse un dump d'un smartphone Android. L'objectif est de mettre en lumière les données sensibles pouvant être extraites d'un smartphone.

🔍 Voici ce qu'il est possible de trouver :

- 📞 Informations de Contacts : Des détails personnels tels que les adresses e-mail, les numéros de téléphone et les noms.

- 📲 Téléchargements et Applications : Les horodatages et les informations sur les applications ont révélé l'historique des téléchargements et les préférences d'applications de l'utilisateur.

- 💼 Données d'Utilisation Personnelle : Les cycles de la batterie, les habitudes d'utilisation des applications et la durée des sessions d'applications.

- 🔐 Mots de Passe WiFi : un mots de passe WiFi est stocké en clair dans les fichiers système.

- 📷 Fichiers : Photos, vidéos et musiques faisaient partie des divers fichiers multimédias accessibles depuis la sauvegarde.

... il y aurait sûrement d'autres données à découvrir !

🤔 La Conclusion : we write-up démontre comment les smartphones stockent une quantité considérable d'informations personnelles. En tant qu'utilisateurs de smartphones, nous devons rester vigilants quant aux données stockées sur nos appareils et prendre des mesures proactives pour protéger notre vie privée.

🔒 Protégez Vos Données : Passez régulièrement en revue et supprimez les données inutiles, utilisez des codes d'accès solides, évitez les connexions à des réseaux WiFi non sécurisés et soyez prudents quant aux applications que vous téléchargez.

🔍 Restez Informés : La sensibilisation est essentielle, comprendre les risques potentiels peut nous aider à prendre des décisions éclairées et à éviter les mauvaises surprises.

#CyberSécurité #ConsciencePrivée #EnquêteSmartphone #ProtectionDesDonnées #Smartphone #BlueTeam

🔗 Lien vers le Rapport Complet : 

---

The Scenario:

> Our IR team took a disk dump of the android phone. As a soc analyst, analyze the dump and answer the provided questions.

I used this online tool to open SQL database : [Sqliteviewer](https://sqliteviewer.app/).

![1896 phone.](/img/write-up/old_phone.jpg)

<figcaption>1896 phone.</figcaption>

## Abstract

To complete the challenge, you have to browse all the given files. Be curious, take the time to understand how the data is organized and use a tool like `grep`.

- contacts informations (mail, phone number, names...)

- downloads timestamps and application informations

- personnal usage information like:

  - battery cycles

  - applications used, when and for how long

- secrets like WiFi password

- medias (photos, videos, music...)

And surrely many other personnal datas such as applications informations, passwords, messages...

## Q1. What is the email address of Zoe Washburne?

`AgentData > contacts3.db`

This file seems to contains the contact datas: `zoewash@0x42.null`. `AgentData` directory seems interesting as it contains general informations.

## Q2. What was the device time in UTC at the time of acquisition? (hh:mm:ss)

`LiveData > device_datetime_utc.txt`

The `LiveData` directory seems to contains general system information related.

answer: 18:17:56

## Q3. What time was Tor Browser downloaded in UTC? (hh:mm:ss)

`AgentData > downloads.db`

The lastmod column provide a timestamp, convert it to UTC format: 1619725346000 = 2021-04-29 19:42:26 UTC.

## Q4. What time did the phone charge to 100% after the last reset? (hh:mm:ss)

There is an interesting file here : `Live Data > Dumpsys Data > batterystats.txt`.

This file logs battery events, we can see the last reset time, and bellow when the battery was fully charged:

```logs
RESET:TIME: 2021-05-21-13-12-19
...
+5m01s459ms (3) 100 status=full charge=2665
```

Adding 5m01s to 13:12:19 gives the answer: 13:17:20.

## Q5. What is the password for the most recently connected WIFI access point?

I didn't found the answer. Thanks to this WU [CyberDefenders: LGDroid](https://forensicskween.com/ctf/cyberdefenders/lgdroid/), I discovered a usefull ressource, the [SANS Smartphone Cheatsheet](https://www.sans.org/posters/dfir-advanced-smartphone-forensics/). According to the poster, username and passwords can be found here: `/data/com.android.providers.settings/`. The `PreSharedKey` tag provides the answer: ThinkingForest!.

## Q6. What app was the user focused on at 2021-05-20 14:13:27?

The question seems to insinuate that the app was running during the acquisition. That's why I searched on "14:13:27" in `Live Data > usage_stats.txt` and found Youtube.

```logs
    time="2021-05-20 14:13:27" type=MOVE_TO_BACKGROUND package=com.lge.launcher3 class=com.lge.launcher3.LauncherExtension flags=0x0 
    time="2021-05-20 14:13:27" type=STANDBY_BUCKET_CHANGED package=com.google.android.youtube standbyBucket=10 reason=u-si flags=0x0 
    time="2021-05-20 14:13:27" type=MOVE_TO_FOREGROUND package=com.google.android.youtube class=com.google.android.apps.youtube.app.application.Shell_HomeActivity flags=0x0 
    time="2021-05-20 14:13:27" type=MOVE_TO_BACKGROUND package=com.google.android.youtube class=com.google.android.apps.youtube.app.application.Shell_HomeActivity flags=0x0 
    time="2021-05-20 14:13:27" type=MOVE_TO_FOREGROUND package=com.google.android.youtube class=com.google.android.apps.youtube.app.watchwhile.WatchWhileActivity flags=0x0 
```

## Q7. How much time did the suspect watch Youtube on 2021-05-20? (hh:mm:ss)

Always in `Live Data > usages_stats.txt`, I searched for `package=com.google.android.youtube` and I found data like this : 

```logs
 In-memory weekly stats
  timeRange="5/20/2021, 11:16 AM ??5/21/2021, 1:17 PM" 
    packages
      package=com.hy.system.fontserver totalTime="00:00" lastTime="1969-12-31 18:00:00" appLaunchCount=0 
      package=com.android.LGSetupWizard totalTime="00:00" lastTime="1969-12-31 18:00:00" appLaunchCount=0 
      package=com.google.android.youtube totalTime="8:34:29" lastTime="2021-05-20 22:47:57" appLaunchCount=1 
```

Unfortunately, 08:34:29 is not the answer. I supposed a rounded error, tried 08:34:**30**, and it worked. I checked the advice provided by the challenge and the attended method was to check the date of the last `MOVE_TO_FOREGROUND` activity and last `MOVE_TO_BACKGROUND` and substract:

MOVE_TO_FOREGROUND package=com.google.android.youtube -> 14:13:27

MOVE_TO_BACKGROUND package=com.google.android.youtube -> 22:47:57

We find 08:34:30, but quite a tricky question, as the field `totalTime` is filled.

## Q8. "suspicious.jpg: What is the structural similarity metric for this image compared to a visually similar image taken with the mobile phone?

The first challenge here is to understand the question. You have to found the second image, it is located in `sdcard > DCIM > Camera`. The structural similarity metric is used to measure visual similarity between two images, this online tool allows me to calculate it: [http://darosh.github.io/image-ms-ssim-js/test/browser_test.html](http://darosh.github.io/image-ms-ssim-js/test/browser_test.html). I found a SSIM of **0.99** which is consisent, as both images are similars.
