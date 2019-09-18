#CRA + Electron + Typescript

####Create React App : 
C'est l'outils officiel (facebook) pour créer des App React :
https://facebook.github.io/create-react-app/
il embarque et repose sur Webpack mais ne permet pas sa configuration sans "ejecter" la config, et on ne veut pas sa car on perd toutes la bonne configuration de react (un paquet permet de modifier la config sans ejecter plus tard).

```
$ npx create-react-app my-app --typescript
$ cd my-app
```
Si on faisait une appli React pour le web on pourrait dev directement ici.

```
$ yarn add electron electron-builder --dev
```

####wait-on :
Ce paquet permet en dev de pipe des process, en l'occurence ça sera le build Webpack qui sera servit sur le port 3000 et le point d'entrée d'electron en dev sera ce port. En prod, le build complet est généré donc il n'y aura pas de différence.

Passer par le build webpack permet le hot reload.
```
$ yarn add wait-on concurrently --dev
```
```
$ yarn add electron-is-dev
$ yarn add cross-env --dev
```

On crée le fichier `public/electron.js` avec : 

```
const electron = require('electron');
const app = electron.app;
const BrowserWindow = electron.BrowserWindow;

const path = require('path');
const isDev = require('electron-is-dev');

let mainWindow;

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 680,
    webPreferences: {
      nodeIntegration: true
    },
  });
  mainWindow.loadURL(isDev ? 'http://localhost:3000' : `file://${path.join(__dirname, '../build/index.html')}`);
  if (isDev) {
    // Open the DevTools.
    //BrowserWindow.addDevToolsExtension('<location to your react chrome extension>');
    mainWindow.webContents.openDevTools();
  }
  mainWindow.on('closed', () => mainWindow = null);
}

app.on('ready', createWindow);

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

app.on('activate', () => {
  if (mainWindow === null) {
    createWindow();
  }
});
```

On ajoute ce script au `package.json`: 

```
"dev": "concurrently \"cross-env BROWSER=none yarn start\" \"wait-on http://localhost:3000 && electron .\"",
```

On attend donc que l'application soit servie sur le port 3000 avant de lancer electron

On ajoute le champ `main`au `package.json`: 

```
"main": "public/electron.js",
```

####Rescripts :

Permet de modifier la config de Webpack sans eject

```
$ yarn add @rescripts/cli @rescripts/rescript-env --dev
```
il faut ensuite changer dans le `package.json` les scripts :

```
"start": "react-scripts start",
"build": "react-scripts build",
"test": "react-scripts test",
```
en
```
"start": "rescripts start",
"build": "rescripts build",
"test": "rescripts test",
```
On créé un nouveau fichier `.rescriptsrc.js` avec : 

```
module.exports = [require.resolve('./.webpack.config.js')]
```
puis un `.webpack.config.js`

```
// define child rescript
module.exports = config => {
  config.target = 'electron-renderer';
  return config;
}
```

Ca permet de configurer webpack pour être conscient que le build tourne sous node et a donc accès aux modules natifs qui ne sont pas présents normalement dans un navigateur.



Par défault, CRA build un index.html qui utilise des path absolus, ce qui plantera avec electron, il faut donc rajouter le champ `homepage` dans le `package.json` : 

```
"homepage": "./",
```

---
###Packaging

On utilise electron builder pour packager l'application mais aussi pour rebuild les dépendances natives dont on a besoin (sqlite3) pour chaque plateforme 

```
$ yarn add electron-builder --dev
```

on rajoute ces 3 scripts dans package.json : 

```
"predist": "yarn build",
"dist": "rm -rf dist/ && electron-builder",
"postinstall": "install-app-deps"
```

`postinstall` sera exécuté après chaque installation de paquet pour rebuild les dépendances natives.
`predist` sera exécuté avant la commande `dist` pour que webpack build l'application en mode production
puis `dist` appelle electron-builder. Sans option (-l / -lmw etc...) il ne build que pour la plateforme en cours.

il faut rajouter les champs suivant dans `package.json`: 

```
"author": {
    "name": "Twinlife",
    "email": "twinlife@twin.life",
    "url": "https://twinlife.com"
  },
  "build": {
    "publish": false,
    "appId": "com.twinlife.twinme-desktop",
    "productName": "Twinme Desktop",
    "copyright": "Copyright © 2019 ${author}",
    "mac": {
      "category": "public.app-category.utilities"
    },
    "files": [
      "build/**/*",
      "node_modules/**/*"
    ],
    "directories": {
      "buildResources": "assets"
    }
  }
```

c'est une config minimale, on peut définir les formats d'application (.deb, .dmg, AppImage, .msi etc...) générés etc...

--- 
###Externals (Webpack)

il faut que Webpack soit conscient que certains packages sont des binaires non natifs de Node.js, c'est le cas de sqlite3

```
$ yarn add better-sqlite3
$ yarn add @types/better-sqlite3 --dev
```

pour pouvoir l'utiliser il faut le définir comme externals dans la config de Webpack, et ce grace à Rescripts. Dans `.webpack.config.js` : 

```
// define child rescript
module.exports = config => {
    config.target = 'electron-renderer';
    config.externals = {
        'better-sqlite3': 'commonjs better-sqlite3'
    };
    return config;
}
```

---

###Usage

Lancer la version de dev avec le hot reload : 
```
$ yarn dev
```
Packager l'application : 
```
$ yarn dist
```