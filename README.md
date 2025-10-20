<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>ChatRB 💬 Grupos con Notificaciones</title>
<style>
body { background-color: #0f172a; color: white; font-family: Arial, sans-serif; margin: 0; padding: 20px; }
h1 { color: #38bdf8; }
#chatContainer { display: flex; width: 90%; max-width: 1000px; gap: 20px; }
#chat { background: #1e293b; flex: 2; height: 400px; border-radius: 10px; padding: 10px; overflow-y: auto; box-shadow: 0 0 10px #0ea5e9;}
#users { background: #1e293b; flex: 1; border-radius: 10px; padding: 10px; height: 400px; overflow-y: auto; box-shadow: 0 0 10px #0ea5e9;}
input, button { padding: 10px; border-radius: 8px; border: none; margin-bottom: 5px;}
#messageInput, #groupCodeInput, #usernameInput { width: 70%; background: #334155; color: white; }
button { background: #38bdf8; color: #0f172a; font-weight: bold; cursor: pointer; }
button:hover { background: #0ea5e9; }
#logoutBtn { margin-top: 10px; background: red; color: white; font-weight: bold; cursor: pointer; }
#namePopup { display: none; position: fixed; top:0; left:0; width:100%; height:100%; background: rgba(0,0,0,0.8); justify-content: center; align-items: center; z-index: 100;}
#namePopupContent { background: #1e293b; padding: 20px; border-radius: 10px; text-align: center;}
#namePopupContent input { width: 80%; background: #334155; color:white; margin-bottom:10px;}
#namePopupContent button { padding: 10px 15px; }
#currentGroup { margin-bottom: 10px; color: #facc15;}
</style>
</head>
<body>

<h1>💬 ChatRB - Grupos con Notificaciones</h1>

<div id="chatContainer">
  <div>
    <div id="currentGroup">No estás en ningún grupo</div>
    <div id="chat"></div>
    <input type="text" id="messageInput" placeholder="Escribe un mensaje..." />
    <button id="sendBtn">Enviar</button>
    <br>
    <input type="text" id="groupCodeInput" placeholder="Código del grupo"/>
    <button id="joinGroupBtn">Unirse al grupo</button>
    <button id="createGroupBtn">Crear grupo</button>
    <br>
    <button id="logoutBtn">Cerrar sesión</button>
  </div>
  <div id="users">
    <h3>Usuarios Conectados</h3>
    <ul id="userList"></ul>
  </div>
</div>

<!-- Popup para elegir nombre -->
<div id="namePopup">
  <div id="namePopupContent">
    <h2>Elige tu nombre</h2>
    <input type="text" id="usernameInput" placeholder="Tu nombre..." />
    <br>
    <button id="setNameBtn">Aceptar</button>
  </div>
</div>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.13.2/firebase-app.js";
import { getFirestore, collection, addDoc, doc, setDoc, query, orderBy, onSnapshot, getDocs, updateDoc, arrayUnion } from "https://www.gstatic.com/firebasejs/10.13.2/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "AIzaSyB8laLYCXZvLj_f5iI0DSBBxy4rCmcWF38",
  authDomain: "chatrb-850cb.firebaseapp.com",
  projectId: "chatrb-850cb",
  storageBucket: "chatrb-850cb.firebasestorage.app",
  messagingSenderId: "952386742311",
  appId: "1:952386742311:web:a0a6cf328be2ee6147c602"
};
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

let username = "";
let currentGroup = "";
let firstLoad = true;

// Solicitar permiso de notificaciones
if ("Notification" in window) {
  Notification.requestPermission();
}

// Mostrar popup de nombre
function mostrarPopup(){ document.getElementById("namePopup").style.display = "flex"; }

// Revisar si ya hay usuario guardado
if(localStorage.getItem("username")){
    username = localStorage.getItem("username");
    document.getElementById("namePopup").style.display = "none";
    registrarUsuario();
} else {
    mostrarPopup();
}

// Función de aceptar nombre
function setNombre(){
  const input = document.getElementById("usernameInput").value.trim();
  if(input){ 
    username = input;
    localStorage.setItem("username", username); // Guardar en localStorage
    document.getElementById("namePopup").style.display = "none"; 
    registrarUsuario();
  }
}
document.getElementById("setNameBtn").addEventListener("click", setNombre);

// Registrar usuario en Firestore
async function registrarUsuario(){
  const userRef = doc(db,"usuarios",username);
  await setDoc(userRef,{online:true});
  // Escuchar usuarios conectados
  const q = query(collection(db,"usuarios"));
  onSnapshot(q,(snapshot)=>{
    const ul = document.getElementById("userList");
    ul.innerHTML="";
    snapshot.forEach(doc=>{
      const data = doc.data();
      ul.innerHTML += `<li>${doc.id} - ${data.online ? "En línea":"Offline"}</li>`;
    });
  });
}

// Enviar mensaje
async function enviarMensaje(){
  const input = document.getElementById("messageInput");
  const texto = input.value.trim();
  if(!texto || !username || !currentGroup) return;
  await addDoc(collection(db,"mensajes"),{
    texto: texto,
    username: username,
    grupo: currentGroup,
    fecha: new Date()
  });
  input.value="";
}

// Crear grupo
async function crearGrupo(){
  const code = prompt("Ingresa un código para el grupo:");
  if(!code) return;
  const grupoRef = doc(db,"grupos",code);
  await setDoc(grupoRef,{miembros:[username]});
  currentGroup = code;
  document.getElementById("currentGroup").textContent = `Grupo: ${currentGroup}`;
  escucharMensajes();
}

// Unirse a grupo
async function unirseGrupo(){
  const code = document.getElementById("groupCodeInput").value.trim();
  if(!code) return;
  const grupoRef = doc(db,"grupos",code);
  const grupoDoc = await (await getDocs(collection(db,"grupos"))).docs.find(d=>d.id===code);
  if(!grupoDoc){ alert("Grupo no existe"); return;}
  await updateDoc(grupoRef,{miembros:arrayUnion(username)});
  currentGroup = code;
  document.getElementById("currentGroup").textContent = `Grupo: ${currentGroup}`;
  escucharMensajes();
}

// Escuchar mensajes del grupo y mostrar notificaciones
function escucharMensajes(){
  if(!currentGroup) return;
  const q = query(collection(db,"mensajes"), orderBy("fecha"));
  onSnapshot(q,(snapshot)=>{
    const chat = document.getElementById("chat");
    chat.innerHTML="";
    snapshot.forEach(doc=>{
      const data = doc.data();
      if(data.grupo===currentGroup){
        chat.innerHTML += `<p><strong>${data.username}:</strong> ${data.texto}</p>`;
        // Notificación si no es nuestro mensaje
        if(!firstLoad && data.username!==username && Notification.permission==="granted"){
          new Notification(`Nuevo mensaje de ${data.username}`, { body: data.texto });
        }
      }
    });
    chat.scrollTop = chat.scrollHeight;
    firstLoad = false;
  });
}

// Eventos
document.getElementById("sendBtn").addEventListener("click", enviarMensaje);
document.getElementById("messageInput").addEventListener("keypress",(e)=>{ if(e.key==="Enter") enviarMensaje(); });
document.getElementById("createGroupBtn").addEventListener("click", crearGrupo);
document.getElementById("joinGroupBtn").addEventListener("click", unirseGrupo);

// Botón cerrar sesión
document.getElementById("logoutBtn").addEventListener("click", ()=>{
    localStorage.removeItem("username");
    location.reload(); // Recarga para volver a mostrar popup
});

</script>
</body>
</html>
