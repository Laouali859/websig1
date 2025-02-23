DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>WebSIG - Surveillance et alerte des Inondations</title>
  
  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.7.1/dist/leaflet.css" />
  <!-- Leaflet JS -->
  <script src="https://unpkg.com/leaflet@1.7.1/dist/leaflet.js"></script>
  <!-- Plugin pour effet 3D (Leaflet.TiltShift) -->
  <script src="https://unpkg.com/leaflet-tiltshift/Leaflet.TiltShift.js"></script>
  
  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #dff6ff;
      margin: 0;
      padding: 0;
      text-align: center;
    }
    h1 {
      color: #00695c;
      margin: 20px;
    }
    .auth-container {
      margin: 20px;
    }
    input, button {
      padding: 10px;
      margin: 5px;
      width: 200px;
    }
    button {
      background-color: #00796b;
      color: white;
      border: none;
      cursor: pointer;
    }
    button:hover {
      background-color: #004d40;
    }
    /* Effet 3D sur le conteneur de la carte */
    #map {
      height: 500px;
      width: 80%;
      margin: auto;
      border: 2px solid #00695c;
      box-shadow: 5px 5px 15px rgba(0, 0, 0, 0.3);
      position: relative;
      transform: perspective(1800px) rotateX(5deg);
      transform-style: preserve-3d;
    }
    /* La légende est cachée par défaut */
    #legend {
      background: rgba(255, 255, 255, 0.9);
      padding: 10px;
      position: absolute;
      bottom: 10px;
      right: 10px;
      z-index: 1000;
      font-size: 14px;
      border-radius: 5px;
      box-shadow: 2px 2px 10px rgba(0,0,0,0.3);
      display: none;
    }
    /* Bouton pour basculer l'affichage de la légende */
    #toggleLegend {
      position: absolute;
      bottom: 10px;
      right: 10px;
      z-index: 1100;
      background: #00796b;
      color: white;
      padding: 5px 10px;
      border: none;
      border-radius: 3px;
      cursor: pointer;
    }
    /* Canvas pour l'effet de précipitations */
    #rainCanvas {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      pointer-events: none;
      z-index: 5000;
    }
  </style>
</head>
<body>
  <h1>🌍 WebSIG - Surveillance et des Inondations</h1>
  
  <!-- Formulaire d'authentification et boutons -->
  <div class="auth-container">
    <input type="email" id="email" placeholder="Email">
    <input type="password" id="password" placeholder="Mot de passe">
    <button onclick="inscrireUtilisateur()">S'inscrire</button>
    <button onclick="connexionUtilisateur()">Se connecter</button>
    <button onclick="deconnexionUtilisateur()">Se déconnecter</button>
    <button onclick="simulerAndroid()">Simuler connexion Android</button>
  </div>
  
  <!-- Carte avec effet de précipitations -->
  <div id="map">
    <canvas id="rainCanvas"></canvas>
  </div>
  
  <!-- Bouton pour afficher/cacher la légende -->
  <button id="toggleLegend">Afficher la légende</button>
  
  <!-- Légende masquée par défaut -->
  <div id="legend">
    <b>Légende</b><br>
    <div style="display:inline-flex;align-items:center;">
      <div style="position: relative; width:48px; height:48px;">
        <img src="https://img.icons8.com/fluency/48/000000/weather-station.png" style="width:48px;height:48px;">
        <img src="https://img.icons8.com/fluency/16/000000/radio-tower.png" style="width:16px;height:16px; position: absolute; top:-4px; left:16px;">
      </div>&nbsp; Capteur IoT (Station avec antenne)
    </div><br>
    <b>Paramètres mesurés :</b><br>
    <img src="https://img.icons8.com/fluency/24/000000/water.png" width="16" height="16"> Niveau d'eau<br>
    <img src="https://img.icons8.com/fluency/24/000000/temperature.png" width="16" height="16"> Température<br>
    <img src="https://img.icons8.com/fluency/24/000000/hygrometer.png" width="16" height="16"> Humidité<br>
    <img src="https://img.icons8.com/fluency/24/000000/barometer.png" width="16" height="16"> Pression<br>
    <img src="https://img.icons8.com/fluency/24/000000/raindrop.png" width="16" height="16"> Précipitations<br>
    <img src="https://img.icons8.com/fluency/24/000000/landscape.png" width="16" height="16"> Altitude<br>
    <b>Alertes :</b><br>
    Débordement du fleuve : Niveau d'eau > 4.0 m (rouge)<br>
    Maisons inondées : Précipitations > 100 mm (bleu)
  </div>
  
  <script>
    /********* Initialisation de la carte *********/
    var map = L.map('map').setView([13.5128, 2.1157], 12);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);
    
    /********* Nouvelle icône composite pour les capteurs *********/
    var sensorIcon = L.divIcon({
      html: '<div style="position: relative; width:48px; height:48px;">' +
              '<img src="https://img.icons8.com/fluency/48/000000/weather-station.png" style="width:48px;height:48px;">' +
              '<img src="https://img.icons8.com/fluency/16/000000/radio-tower.png" style="width:16px;height:16px; position: absolute; top:-4px; left:16px;">' +
            '</div>',
      iconSize: [48, 48],
      className: ''
    });
    
    /********* Définition des 4 capteurs installés le long du fleuve *********/
    var sensorsData = [
      {
        id: 1,
        name: "Capteur 1 - Bord du Fleuve",
        lat: 13.485027,
        lng: 2.115111,
        altitude: 5.0,
        waterLevel: 0,
        temperature: 0,
        humidity: 0,
        pressure: 0,
        precipitation: 0,
        alertOverlayRiver: null,
        alertOverlayFlood: null
      },
      {
        id: 2,
        name: "Capteur 2 - Bord du Fleuve",
        lat: 13.518748,
        lng: 2.090952,
        altitude: 5.5,
        waterLevel: 0,
        temperature: 0,
        humidity: 0,
        pressure: 0,
        precipitation: 0,
        alertOverlayRiver: null,
        alertOverlayFlood: null
      },
      {
        id: 3,
        name: "Capteur 3 - Bord du Fleuve",
        lat: 13.531022,
        lng: 2.042149,
        altitude: 4.5,
        waterLevel: 0,
        temperature: 0,
        humidity: 0,
        pressure: 0,
        precipitation: 0,
        alertOverlayRiver: null,
        alertOverlayFlood: null
      },
      {
        id: 4,
        name: "Capteur 4 - Bord du Fleuve",
        lat: 13.498162,
        lng: 2.120429,
        altitude: 6.0,
        waterLevel: 0,
        temperature: 0,
        humidity: 0,
        pressure: 0,
        precipitation: 0,
        alertOverlayRiver: null,
        alertOverlayFlood: null
      }
    ];
    
    sensorsData.forEach(function(sensor) {
      sensor.marker = L.marker([sensor.lat, sensor.lng], { icon: sensorIcon }).addTo(map);
      sensor.marker.bindPopup(generatePopupContent(sensor));
    });
    
    function generatePopupContent(sensor) {
      return `<b>${sensor.name}</b><br>
        <img src="https://img.icons8.com/fluency/24/000000/water.png" width="16" height="16"> Niveau d'eau : ${sensor.waterLevel.toFixed(1)} m<br>
        <img src="https://img.icons8.com/fluency/24/000000/temperature.png" width="16" height="16"> Température : ${sensor.temperature.toFixed(1)}°C<br>
        <img src="https://img.icons8.com/fluency/24/000000/hygrometer.png" width="16" height="16"> Humidité : ${sensor.humidity.toFixed(0)}%<br>
        <img src="https://img.icons8.com/fluency/24/000000/barometer.png" width="16" height="16"> Pression : ${sensor.pressure.toFixed(0)} hPa<br>
        <img src="https://img.icons8.com/fluency/24/000000/raindrop.png" width="16" height="16"> Précipitations : ${sensor.precipitation.toFixed(1)} mm<br>
        <img src="https://img.icons8.com/fluency/24/000000/landscape.png" width="16" height="16"> Altitude : ${sensor.altitude.toFixed(1)} m`;
    }
    
    function updateSensors() {
      sensorsData.forEach(function(sensor) {
        sensor.waterLevel = getRandom(3.0, 6.0);
        sensor.temperature = getRandom(25, 35);
        sensor.humidity = getRandom(60, 90);
        sensor.pressure = getRandom(1008, 1020);
        sensor.precipitation = getRandom(0, 150);
    
        sensor.marker.setPopupContent(generatePopupContent(sensor));
    
        if(sensor.waterLevel > 4.0) {
          if(!sensor.alertOverlayRiver) {
            sensor.alertOverlayRiver = L.circle([sensor.lat, sensor.lng], {
              color: "red",
              fillColor: "#f03",
              fillOpacity: 0.5,
              radius: 300
            }).addTo(map)
              .bindPopup(`<b>Débordement du fleuve</b><br>Niveau d'eau : ${sensor.waterLevel.toFixed(1)} m`);
          } else {
            sensor.alertOverlayRiver.setStyle({ color: "red", fillColor: "#f03" });
            sensor.alertOverlayRiver.setPopupContent(`<b>Débordement du fleuve</b><br>Niveau d'eau : ${sensor.waterLevel.toFixed(1)} m`);
          }
        } else {
          if(sensor.alertOverlayRiver) {
            map.removeLayer(sensor.alertOverlayRiver);
            sensor.alertOverlayRiver = null;
          }
        }
    
        if(sensor.precipitation > 100) {
          if(!sensor.alertOverlayFlood) {
            sensor.alertOverlayFlood = L.circle([sensor.lat, sensor.lng], {
              color: "blue",
              fillColor: "#00f",
              fillOpacity: 0.3,
              radius: 300
            }).addTo(map)
              .bindPopup(`<b>Maisons inondées</b><br>Précipitations : ${sensor.precipitation.toFixed(1)} mm`);
          } else {
            sensor.alertOverlayFlood.setStyle({ color: "blue", fillColor: "#00f" });
            sensor.alertOverlayFlood.setPopupContent(`<b>Maisons inondées</b><br>Précipitations : ${sensor.precipitation.toFixed(1)} mm`);
          }
        } else {
          if(sensor.alertOverlayFlood) {
            map.removeLayer(sensor.alertOverlayFlood);
            sensor.alertOverlayFlood = null;
          }
        }
      });
    }
    
    function getRandom(min, max) {
      return Math.random() * (max - min) + min;
    }
    
    setInterval(updateSensors, 5000);
    updateSensors();
    
    // Toggle de la légende
    document.getElementById('toggleLegend').addEventListener('click', function() {
      var legend = document.getElementById('legend');
      if(legend.style.display === 'none' || legend.style.display === '') {
        legend.style.display = 'block';
        this.textContent = 'Cacher la légende';
      } else {
        legend.style.display = 'none';
        this.textContent = 'Afficher la légende';
      }
    });
    
    /********* Effet de précipitations (pluie) *********/
    var rainCanvas = document.getElementById('rainCanvas');
    var rainCtx = rainCanvas.getContext('2d');
    
    function resizeRainCanvas() {
      rainCanvas.width = document.getElementById('map').offsetWidth;
      rainCanvas.height = document.getElementById('map').offsetHeight;
    }
    resizeRainCanvas();
    window.addEventListener('resize', resizeRainCanvas);
    
    var raindrops = [];
    var numDrops = 200;
    for (var i = 0; i < numDrops; i++) {
      raindrops.push({
        x: Math.random() * rainCanvas.width,
        y: Math.random() * rainCanvas.height,
        length: Math.random() * 20 + 10,
        speed: Math.random() * 4 + 4
      });
    }
    
    function animateRain() {
      rainCtx.clearRect(0, 0, rainCanvas.width, rainCanvas.height);
      rainCtx.strokeStyle = 'rgba(0,162,232,0.7)';
      rainCtx.lineWidth = 1.5;
      rainCtx.lineCap = 'round';
      for (var i = 0; i < numDrops; i++) {
        var drop = raindrops[i];
        rainCtx.beginPath();
        rainCtx.moveTo(drop.x, drop.y);
        rainCtx.lineTo(drop.x, drop.y + drop.length);
        rainCtx.stroke();
        drop.y += drop.speed;
        if (drop.y > rainCanvas.height) {
          drop.y = -drop.length;
          drop.x = Math.random() * rainCanvas.width;
        }
      }
      requestAnimationFrame(animateRain);
    }
    animateRain();
    
    /********* Fonction de simulation de connexion Android *********/
    function simulerAndroid() {
      // Coordonnées simulées du domicile de l'utilisateur (centre de la carte)
      var lat = 13.5128, lon = 2.1157;
      // Vérification des alertes parmi les capteurs
      var alerteDetectee = sensorsData.some(sensor => sensor.waterLevel > 4.0 || sensor.precipitation > 100);
      var popupMsg = `<b>Utilisateur Android</b><br>Connexion simulée depuis votre domicile.<br>`;
      popupMsg += alerteDetectee ? `<span style="color:red;">Alerte détectée</span>` : `<span style="color:green;">Aucune alerte</span>`;
      
      var androidIcon = L.divIcon({
        html: '<img src="https://img.icons8.com/color/48/000000/android-os.png" width="48" height="48">',
        iconSize: [48, 48],
        className: ''
      });
      
      L.marker([lat, lon], { icon: androidIcon }).addTo(map)
        .bindPopup(popupMsg).openPopup();
    }
    
    /********* Fonctions d'authentification et géolocalisation (optionnel) *********/
    function inscrireUtilisateur() {
      var email = document.getElementById('email').value;
      var password = document.getElementById('password').value;
      if (!email || !password) {
        alert("Veuillez remplir tous les champs");
        return;
      }
      if (localStorage.getItem(email)) {
        alert("Utilisateur déjà inscrit");
      } else {
        localStorage.setItem(email, password);
        alert("Inscription réussie");
      }
    }
    function connexionUtilisateur() {
      var email = document.getElementById('email').value;
      var password = document.getElementById('password').value;
      if (!email || !password) {
        alert("Veuillez remplir tous les champs");
        return;
      }
      var storedPassword = localStorage.getItem(email);
      if (storedPassword && storedPassword === password) {
        localStorage.setItem('currentUser', email);
        alert("Connexion réussie");
        if (navigator.geolocation) {
          navigator.geolocation.getCurrentPosition((position) => {
            var lat = position.coords.latitude;
            var lon = position.coords.longitude;
            var userIcon = L.divIcon({
              html: '<img src="https://img.icons8.com/fluency/48/000000/marker.png" width="48" height="48">',
              iconSize: [48, 48],
              className: ''
            });
            L.marker([lat, lon], { icon: userIcon }).addTo(map)
              .bindPopup(`<b>Votre position</b>`).openPopup();
          });
        }
      } else {
        alert("Email ou mot de passe incorrect");
      }
    }
    function deconnexionUtilisateur() {
      localStorage.removeItem('currentUser');
      alert("Déconnexion réussie");
    }
  </script>
</body>
</html>
