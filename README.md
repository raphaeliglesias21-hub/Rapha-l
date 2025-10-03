[openfront_mobile_admob.html](https://github.com/user-attachments/files/22674615/openfront_mobile_admob.html)
<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <title>Chat Programmes D</title>
  <style>
    :root{
      --bg:#f5f7fb;
      --card:#ffffff;
      --accent:#5865F2;
      --muted:#64748b;
      --me:#dfe7ff;
      --other:#ffffff;
      --radius:14px;
    }
    *{box-sizing:border-box}
    body{margin:0;font-family:Inter, system-ui, Arial; background:var(--bg); color:#0b1220}
    .wrap{max-width:980px;margin:28px auto;padding:16px}
    header{display:flex;align-items:center;justify-content:space-between;padding:12px 0}
    header h1{margin:0;font-size:20px}
    .card{background:var(--card);border-radius:16px;padding:18px;box-shadow:0 6px 20px rgba(8,15,31,0.06)}
    .chat-area{height:520px;overflow:auto;padding:18px;border-radius:12px;background:linear-gradient(180deg,#fbfdff,#f8f9ff);border:1px solid #eef2ff}
    .bubbles{display:flex;flex-direction:column;gap:10px}
    .bubble{max-width:78%;padding:10px 14px;border-radius:16px;box-shadow:0 2px 6px rgba(15,23,42,0.04);word-break:break-word}
    .bubble.me{margin-left:auto;background:var(--me)}
    .bubble.other{margin-right:auto;background:var(--other)}
    .meta{font-size:12px;color:var(--muted);margin-bottom:6px}
    .controls{display:flex;gap:10px;align-items:center;margin-top:12px}
    .controls input[type="text"]{flex:0 0 160px;padding:10px;border-radius:10px;border:1px solid #e6eefb;background:#fff}
    .controls input[type="text"].msg{flex:1;padding:12px}
    .btn{background:var(--accent);color:white;padding:10px 14px;border-radius:10px;text-decoration:none;border:none;cursor:pointer;font-weight:700}
    .small{font-size:13px;color:var(--muted);text-align:center;margin-top:10px}
    .topbar{display:flex;gap:12px;align-items:center}
    .badge{background:#eef2ff;color:var(--accent);padding:6px 10px;border-radius:999px;font-weight:700}
    @media (max-width:760px){
      .chat-area{height:420px}
      .controls{flex-direction:column;align-items:stretch}
      .controls input[type="text"]{width:100%}
    }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Programmes D — Chat</h1>
      <div class="topbar">
        <div class="badge">Public</div>
      </div>
    </header>

    <div class="card" style="margin-top:12px">
      <div style="display:flex;gap:20px;align-items:flex-start">
        <div style="flex:1">
          <div class="chat-area" id="chatArea" aria-live="polite">
            <div class="bubbles" id="messages"></div>
          </div>

          <div class="small">Règles : pas d'insultes, pas de spam, pas de contenus illégaux. Messages publics.</div>

          <div class="controls" style="margin-top:10px">
            <input id="nameInput" type="text" placeholder="Ton pseudo (ex : Arthur)" maxlength="20">
            <input id="msgInput" class="msg" type="text" placeholder="Écris ton message..." maxlength="500">
            <button id="sendBtn" class="btn">Envoyer</button>
          </div>
        </div>

        <aside style="width:260px">
          <div style="display:flex;flex-direction:column;gap:12px">
            <div class="card" style="padding:12px;background:linear-gradient(180deg,#fff,#fbfdff)">
              <strong>Infos</strong>
              <p style="margin:8px 0 0;color:var(--muted);font-size:14px">
                Chat public intégré — fonctionne sur mobile et desktop. Les messages s'affichent en temps réel.
              </p>
            </div>

            <div class="card" style="padding:12px;text-align:center">
              <strong>Nettoyage automatique</strong>
              <p style="margin:8px 0 0;color:var(--muted);font-size:13px">
                Tu peux activer une rotation des messages dans Firebase si tu veux expirer les anciens.
              </p>
            </div>
          </div>
        </aside>
      </div>
    </div>
  </div>

  <!-- Firebase (compat) -->
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-database-compat.js"></script>

  <script>
    // --- REMPLACE ICI TA CONFIG FIREBASE (copie depuis la console Firebase) ---
    const firebaseConfig = {
      apiKey: "REPLACE_API_KEY",
      authDomain: "REPLACE_AUTHDOMAIN",
      databaseURL: "https://REPLACE_PROJECT.firebaseio.com",
      projectId: "REPLACE_PROJECTID",
      storageBucket: "REPLACE_BUCKET.appspot.com",
      messagingSenderId: "REPLACE_SENDER_ID",
      appId: "REPLACE_APP_ID"
    };
    // -----------------------------------------------------------------------

    // Initialisation
    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();
    const messagesRef = db.ref('public_chat/messages');

    // Utilitaires
    function escapeHtml(s){
      return s
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#039;');
    }

    function timeAgo(ts){
      const d = new Date(ts);
      return d.toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'});
    }

    // Lecture en temps réel : quand un message est ajouté
    messagesRef.limitToLast(200).on('child_added', snapshot => {
      const data = snapshot.val();
      if(!data) return;
      addMessageToDOM(data);
    });

    // Évite les doublons si on reload et récupère les anciens messages : on va rendre par ordre
    messagesRef.limitToLast(200).once('value').then(snapshot => {
      const msgs = [];
      snapshot.forEach(child => msgs.push(child.val()));
      msgs.sort((a,b) => a.ts - b.ts);
      // on vide et réaffiche proprement
      const container = document.getElementById('messages');
      container.innerHTML = '';
      msgs.forEach(addMessageToDOM);
      scrollToBottom();
    });

    function addMessageToDOM(msg){
      const container = document.getElementById('messages');
      // protection basique
      const name = escapeHtml(msg.name || 'Anonyme');
      const text = escapeHtml(msg.text || '');
      const ts = msg.ts || Date.now();
      const me = (localStorage.getItem('chat_name') === name);
      const div = document.createElement('div');
      div.className = 'bubble ' + (me ? 'me' : 'other');
      div.innerHTML = '<div class="meta"><strong>'+name+'</strong> · <span style="color:var(--muted)">'+timeAgo(ts)+'</span></div>'
                    + '<div class="text">'+text+'</div>';
      container.appendChild(div);
      scrollToBottom();
    }

    function scrollToBottom(){
      const area = document.getElementById('chatArea');
      area.scrollTop = area.scrollHeight;
    }

    // Envoi de message
    const sendBtn = document.getElementById('sendBtn');
    const nameInput = document.getElementById('nameInput');
    const msgInput = document.getElementById('msgInput');

    // restore name if déjà choisi
    if(localStorage.getItem('chat_name')){
      nameInput.value = localStorage.getItem('chat_name');
    }

    function sendMessage(){
      const name = (nameInput.value || 'Anonyme').trim().slice(0,20);
      const text = msgInput.value.trim().slice(0,500);
      if(!text) return;
      // stockage local du nom
      localStorage.setItem('chat_name', name);

      // anti-spam simple : limiter la longueur et le rythme
      sendBtn.disabled = true;
      setTimeout(()=> sendBtn.disabled = false, 800);

      // push dans Firebase
      const payload = {
        name: name,
        text: text,
        ts: Date.now()
      };
      messagesRef.push(payload).then(() => {
        msgInput.value = '';
      }).catch(err => {
        alert('Erreur en envoyant le message : ' + err.message);
      });
    }

    // envoi au clic et Enter
    sendBtn.addEventListener('click', sendMessage);
    msgInput.addEventListener('keydown', e => { if(e.key === 'Enter') sendMessage(); });

    // scroll sur focus du message
    msgInput.addEventListener('focus', scrollToBottom);
  </script>
</body>
</html>
