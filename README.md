# test
repo pour tester la librairie JS Git-simple

**test push sucessful yo**

```typescript

import { app, BrowserWindow, dialog, ipcMain, Notification } from "electron";
import * as path from "path";
import * as fs from "fs";
import * as url from "url";
import { Path } from "../bridge/app";
import simpleGit, { SimpleGit, DiffResult } from "simple-git";

let shouldQuit = false;

let mainWindow: BrowserWindow;
let root: string;

function createWindow(): BrowserWindow {
mainWindow = new BrowserWindow({
width: 1400,
height: 900,
//frame: false,
webPreferences: {
nodeIntegration: true,
preload: path.join(__dirname, `preload.js`),
},
});

//mainWindow.loadURL("http://localhost:4200");
mainWindow.loadURL(
url.format({
pathname: path.join(__dirname, `../angular/dist/index.html`),
protocol: "file:",
slashes: true,
})
);

mainWindow.on('close', (event) => {
if (!shouldQuit) {
event.preventDefault(); // Prevent the window from closing immediately
showQuitConfirmationDialog();
}
});
// Open the DevTools.
mainWindow.webContents.openDevTools()

return mainWindow;
}

app.whenReady().then(() => {
initHandle();
createWindow();
app.on("activate", () => {
if (BrowserWindow.getAllWindows().length === 0) createWindow();
});
});

app.on('window-all-closed', function () {
if (process.platform !== 'darwin') app.quit()
})

function showQuitConfirmationDialog() {
const options = {
type: "question",
buttons: ["Quit", "Cancel"],
defaultId: 1,
title: "Confirmation",
message: "You have unsaved changes, are you sure you want to quit?",
} as Electron.MessageBoxOptions;

dialog.showMessageBox(mainWindow, options).then((response) => {
const buttonIndex = response.response;
if (buttonIndex === 0) {
shouldQuit = true;
app.quit();
}
});
}

function initHandle() {
ipcMain.handle('app-exit', exit);
ipcMain.handle('save-file', (_event, content, path) => save(content, path));
ipcMain.handle('path', async () => searchPath(root = await openDialog({ properties: ['openDirectory' ] }), [".md"]));
ipcMain.handle('reloadpath', async (_event, path) => searchPath(path, [".md"]));
ipcMain.handle('deleteFile', async (_event, path) => deleteFile(path));
ipcMain.handle('deleteFolder', async (_event, path) => deleteFolder(path));
ipcMain.handle('picturePath', async () => searchPath(await openDialog({}), [".png"]));
ipcMain.handle('data', (_event, val) => data(val));
ipcMain.handle('createFile', async (_event, path) => await createInterface(path,`file://${__dirname}/inputFile.html`, newFile));
ipcMain.handle('createFolder', async (_event, path) => await createInterface(path,`file://${__dirname}/inputFolder.html`, newFolder));
ipcMain.handle("git-diff", async (_event, val) => gitDiff(val));
ipcMain.handle("checkIfSaved", (_event, saveStatus) => (shouldQuit = saveStatus))
ipcMain.handle('git-add',async (_event, path) => gitAdd(path.substring(0, path.lastIndexOf('/')),path.substring(path.lastIndexOf('/') + 1)))
ipcMain.handle('git-reset',async (_event, path) => gitReset(path.substring(0, path.lastIndexOf('/')),path.substring(path.lastIndexOf('/') + 1)))
ipcMain.handle('gitcommit', async (_event) => await createInterfaceCommit());
ipcMain.handle('git-push',async (_event,) => await createInterfacePush());
}

function data(filePath: string): Promise<string> {
return new Promise((resolve, reject) => {
fs.readFile(filePath, "utf8", (error, data) => {
if (error) {
reject(error);
return;
}
resolve(data);
});
});
}

async function deleteFile(path: string): Promise<Path | null> {
return new Promise<Path | null>((resolve, reject) => {
fs.unlink(path, (error) => {
if (error) {
reject(error);
} else {
resolve(searchPath(root, [".md"]));
}
});
});
}

async function deleteFolder(path: string): Promise<Path | null> {
return new Promise<Path | null>((resolve, reject) => {
if (!fs.existsSync(path)) {
resolve(searchPath(root, [".md"])); // Le dossier n'existe pas, donc considéré comme supprimé
return;
}

fs.readdirSync(path).forEach((file) => {
const filePath = `${path}/${file}`;

if (fs.lstatSync(filePath).isDirectory()) {
deleteFolder(filePath); // Appeler récursivement pour supprimer les sous-dossiers
} else {
fs.unlinkSync(filePath); // Supprimer le fichier
}
});

fs.rmdir(path, (error) => {
if (error) {
reject(error);
} else {
resolve(searchPath(root, [".md"]));
}
});
});
}


async function newFile(filePath: string, extention?: string[]): Promise<void> {
return new Promise<void>((resolve, reject) => {
if(extention?.includes(path.extname(filePath))){
fs.writeFile(filePath, '', (error) => {
if (error) {
reject(error);
} else {
gitAdd(filePath.substring(0, filePath.lastIndexOf('/')),filePath.substring(filePath.lastIndexOf('/') + 1))
console.log('Le fichier a été créé avec succès.');
resolve();
}
});
}else{
resolve();
}
});
}


async function newFolder(folderPath: string): Promise<void> {
return new Promise<void>((resolve, reject) => {
fs.mkdir(folderPath, { recursive: true }, (error) => {
if (error) {
reject(error);
} else {
console.log('Le dossier a été créé avec succès.');
resolve();
}
});
});
}

async function createInterface(currPath: string, url: string, fct: (path: string, extention?: string[]) => Promise<void>): Promise<Path | null>  {
return new Promise<Path | null>((resolve, reject) => {
const inputWindow = new BrowserWindow({
parent: mainWindow,
modal: true,
show: false,
frame: false,
width: 300,
height: 100,
resizable: false,
skipTaskbar: true,
roundedCorners: true,
webPreferences: {
contextIsolation: true, // Enable context isolation
preload: `${__dirname}/preload.js`, // Preload script to provide IPC communication in the input window
},
});
if(url.includes("inputFile")){
ipcMain.handleOnce('fileName', async (_event, inputValue) => {
await fct(path.join(currPath, inputValue), [".md"])
resolve(searchPath(root, [".md"]));
});
}else{
ipcMain.handleOnce('folderName', async (_event, inputValue) => {
await fct(path.join(currPath, inputValue))
resolve(searchPath(root, [".md"]));
});
}

inputWindow.loadURL(url);


inputWindow.webContents.once('did-finish-load', () => {
inputWindow.show();
inputWindow.focus();
});

inputWindow.on('closed', () => {
inputWindow.destroy();
});
});
}


async function openDialog(options: any): Promise<string>{
const { canceled, filePaths } = await dialog.showOpenDialog(mainWindow, options)
if (!canceled) {
return filePaths[0];
}
return "";
}

async function searchPath(directoryOrFile: string, extention: string[]): Promise<Path | null> {
const stats = await fs.promises.stat(directoryOrFile);

const arbo: Path = new Path();
arbo.name = directoryOrFile;
arbo.children = [];

if (stats.isFile()) {
if (extention.includes(path.extname(directoryOrFile))) {
arbo.children = null;
return arbo;
} else {
return null;
}
} else if (stats.isDirectory()) {
if (!arbo.name.endsWith('/')) {
arbo.name += '/';
}
const files = await fs.promises.readdir(directoryOrFile);

const directories: Path[] = [];
const mdFiles: Path[] = [];

for (const file of files) {
const filePath = path.join(directoryOrFile, file);
const child = await searchPath(filePath, extention);

if (child) {
if (child.children) {
directories.push(child);
} else if (extention.includes(path.extname(file))) {
mdFiles.push(child);
}
}
}

directories.sort((a, b) => a.name.localeCompare(b.name));
mdFiles.sort((a, b) => a.name.localeCompare(b.name));

arbo.children = directories.concat(mdFiles);
}

return arbo;
}

function exit() {
app.exit();
}

function showNotification(NOTIFICATION_TITLE, NOTIFICATION_BODY) {
new Notification({title: NOTIFICATION_TITLE, body: NOTIFICATION_BODY}).show();
}

async function save(content: string, pathToSave: string): Promise<string> {
try {
await fs.promises
.writeFile(pathToSave, content);
showNotification("Sauvegarde", "effectuée");
return "Doc. written";
} catch (err) {
console.error(err);
showNotification("Erreur lors de la sauvegarde", err);
throw err;
}
}



async function gitDiff(path: string): Promise<string[]> {
try {
console.log("Je suis dans gitDiff");
console.log(path)
const git: SimpleGit = simpleGit(path);

// Get the diff between the working directory and the index
const diff: string = await git.diff(["--name-only"]);
const lines: string[] = diff.split("\n");
console.log(lines);
return lines;
} catch (error) {
showNotification("Git Error","This folder is not a git repository, launch git init and commit to initiate it")
console.error('Error executing git diff:', error);
throw error;
}
}

async function gitAdd(path: string, filename: string): Promise<void> {
try {
const git: SimpleGit = simpleGit(path);
//console.log("je suis dans le gitadd")
//console.log(path)
//console.log(filename)
// Specify the file(s) or pattern(s) you want to add
await git.add(filename);

console.log('Files added successfully.');
} catch (error) {
console.error('Error adding files:', error);

}
}

async function gitReset(path: string, filename: string): Promise<void> {
try {
const git: SimpleGit = simpleGit(path);
console.log("je suis dans le gitreset")
console.log(path)
console.log(filename)
// Perform the reset operation
await git.reset([filename]);

console.log('Reset operation completed successfully.');
} catch (error) {
console.error('Error performing reset operation:', error);
}
}

async function gitCommit(path: string, commitmessage: string): Promise<boolean> {
return new Promise<boolean>(async (resolve, reject)=>{
try {
const git: SimpleGit = simpleGit(path);

// Perform the commit operation
await git.commit(commitmessage);

console.log('Commit operation completed successfully.');
resolve(true)
} catch (error) {
console.error('Error performing commit operation:', error);
reject(error)
}

})


}

async function createInterfaceCommit()  {
return new Promise<Path | null>((resolve, reject) => {
const inputWindow = new BrowserWindow({
parent: mainWindow,
modal: true,
show: false,
frame: false,
width: 300,
height: 100,
resizable: false,
skipTaskbar: true,
roundedCorners: true,
webPreferences: {
contextIsolation: true, // Enable context isolation
preload: `${__dirname}/preload.js`, // Preload script to provide IPC communication in the input window
},
});

ipcMain.handleOnce('git-commit', async (_event, inputValue) => {

if(await gitCommit(root,inputValue)){
resolve(searchPath(root,[".md"]))
}
else{
reject(new Error("erreur commit"))
}
})
inputWindow.loadURL(`file://${__dirname}/inputcommit.html`);


inputWindow.webContents.once('did-finish-load', () => {
inputWindow.show();
inputWindow.focus();
});

inputWindow.on('closed', () => {
inputWindow.destroy();
});
});
}

async function gitPush(path: string, username: string, password: string){
let pathbis=path.substring(path.lastIndexOf('/') + 1);
try {
console.log(path)
const git: SimpleGit = simpleGit(path);
let url_git = "https://" + username + ":" + password + "@github.com/" + username + "/" + pathbis + ".git"
await git.push(['--all',url_git]);
console.log('Pushed to remote repository successfully.');
} catch (error) {
console.error('Failed to push to remote repository:', error);
}

}

async function createInterfacePush()  {

const inputWindow = new BrowserWindow({
parent: mainWindow,
modal: true,
show: false,
frame: false,
width: 700,
height: 200,
resizable: false,
skipTaskbar: true,
roundedCorners: true,
webPreferences: {
contextIsolation: true, // Enable context isolation
preload: `${__dirname}/preload.js`, // Preload script to provide IPC communication in the input window
},
});

ipcMain.handleOnce('gitpush', (_event,userValue, passwordValue ) => {

gitPush(root,userValue,passwordValue)
})
inputWindow.loadURL(`file://${__dirname}/push_auth.html`);


inputWindow.webContents.once('did-finish-load', () => {
inputWindow.show();
inputWindow.focus();
});

inputWindow.on('closed', () => {
inputWindow.destroy();
});

}
```
