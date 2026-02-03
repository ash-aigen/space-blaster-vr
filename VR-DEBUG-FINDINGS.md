# VR Ping Pong - Debug Findings: Immersive VR Tracking Issue

**Tutkittu:** 2026-02-03  
**Ongelma:** Peli toimii 2D-tilassa ja kun VR menee taustalle (HOME-nappi), mutta EI immersive VR -tilassa.

---

## 🔴 PÄÄONGELMA: Kaksi kriittistä bugia

### 1. A-Frame 1.5.0 + oculus-touch-controls EI TUE Quest 3:a!

**Dokumentaatiosta:**
- `oculus-touch-controls` (v1.5.0): "Oculus Quest 1 and 2" - **Quest 3 puuttuu!**
- `meta-touch-controls` (v1.7.0): "Oculus Quest 1, 2, 3 and 3s" - **Quest 3 tuettu!**

**Lähde:** https://aframe.io/docs/1.5.0/components/oculus-touch-controls.html vs https://aframe.io/docs/1.7.0/components/meta-touch-controls.html

**Tunnettu issue:** https://github.com/aframevr/aframe/issues/5143
> "The main blocker is that it doesn't set `iterateControllerProfiles: true` in tracked-controls"

Quest 3:n kontrollerit käyttävät eri WebXR-profiilia (`meta-quest-touch-plus`) jota vanha oculus-touch-controls ei tunnista!

### 2. getWorldPosition() palauttaa VÄÄRÄN arvon VR-tilassa!

**Three.js WebXR -bugi** joka vaikuttaa A-Frameen:

Kun VR-sessio on aktiivinen, Three.js kopioi kameran/kontrollerin **maailmakoordinaatit** suoraan `position`-arvoon. Jos objektilla on parent (kuten vrPaddle on kontrollerin lapsi), `getWorldPosition()` laskee parent-transformaation **UUDELLEEN**, jolloin:

```
Todellinen sijainti: (0.5, 1.2, 0.8)
getWorldPosition() palauttaa: (1.0, 2.4, 1.6)  // VÄÄRIN - kerrottu 2x!
```

**Lähteet:**
- https://github.com/mrdoob/three.js/issues/23597
- https://github.com/mrdoob/three.js/issues/16382
- https://github.com/aframevr/aframe/issues/4568

---

## 🟢 RATKAISUT

### Ratkaisu 1: Päivitä A-Frame versioon 1.7.0 + meta-touch-controls

```html
<!-- VANHA (1.5.0) - EI TOIMI Quest 3:lla -->
<script src="https://aframe.io/releases/1.5.0/aframe.min.js"></script>
<a-entity oculus-touch-controls="hand: right"></a-entity>

<!-- UUSI (1.7.0) - TOIMII Quest 3:lla -->
<script src="https://aframe.io/releases/1.7.0/aframe.min.js"></script>
<a-entity meta-touch-controls="hand: right"></a-entity>
```

### Ratkaisu 2: Korjaa getWorldPosition() VR-tilassa

**ÄLKÄÄ KÄYTTÄKÖ näin:**
```javascript
paddle.object3D.getWorldPosition(pos);  // VÄÄRIN VR:ssä!
```

**KÄYTTÄKÄÄ näin:**
```javascript
// Vaihtoehto A: Käytä matrixWorld suoraan
pos.setFromMatrixPosition(paddle.object3D.matrixWorld);

// Vaihtoehto B: Jos matrixWorld ei ole päivittynyt, pakota päivitys
paddle.object3D.updateMatrixWorld(true);
pos.setFromMatrixPosition(paddle.object3D.matrixWorld);
```

### Ratkaisu 3: Älä käytä child-elementtiä - kiinnitä paddle suoraan kontrolleriin

**ONGELMA: Paddle on kontrollerin child**
```html
<a-entity id="rhand" oculus-touch-controls="hand: right">
  <a-entity id="vr-paddle" position="0 0 -0.05">  <!-- Child = ongelma! -->
    ...
  </a-entity>
</a-entity>
```

**PAREMPI: Käytä komponenttia joka seuraa kontrolleria**
```javascript
AFRAME.registerComponent('follow-controller', {
  schema: { target: { type: 'selector' } },
  tick: function() {
    if (!this.data.target) return;
    const controller = this.data.target.object3D;
    const paddle = this.el.object3D;
    
    // Kopioi kontrollerin world matrix
    controller.updateMatrixWorld(true);
    paddle.position.setFromMatrixPosition(controller.matrixWorld);
    paddle.quaternion.setFromRotationMatrix(controller.matrixWorld);
    
    // Lisää offset
    const offset = new THREE.Vector3(0, 0, -0.05);
    offset.applyQuaternion(paddle.quaternion);
    paddle.position.add(offset);
  }
});
```

---

## 📋 KORJATTU KOODI (Suositellut muutokset)

### index.html muutokset:

```html
<!-- 1. Päivitä A-Frame -->
<script src="https://aframe.io/releases/1.7.0/aframe.min.js"></script>

<!-- 2. Vaihda meta-touch-controls -->
<a-entity id="rhand" 
          meta-touch-controls="hand: right"
          laser-controls="hand: right">
```

### JavaScript muutokset:

```javascript
// VANHA - EI TOIMI VR:ssä
function getPaddleWorldPos() {
  const pos = new THREE.Vector3();
  paddle.object3D.getWorldPosition(pos);  // BUG!
  return pos;
}

// UUSI - TOIMII VR:ssä
function getPaddleWorldPos() {
  const pos = new THREE.Vector3();
  
  if (inVR && rhand && rhand.object3D) {
    // VR: Käytä kontrollerin matrixWorld suoraan
    rhand.object3D.updateMatrixWorld(true);
    pos.setFromMatrixPosition(rhand.object3D.matrixWorld);
    
    // Lisää paddle offset kontrollerin koordinaateissa
    const offset = new THREE.Vector3(0, 0, -0.05);
    const quaternion = new THREE.Quaternion();
    quaternion.setFromRotationMatrix(rhand.object3D.matrixWorld);
    offset.applyQuaternion(quaternion);
    pos.add(offset);
  } else {
    // Desktop: Normaali tapa toimii
    desktopPaddle.object3D.getWorldPosition(pos);
  }
  
  return pos;
}
```

---

## 🔍 MIKSI SE TOIMII KUN PAINAA HOME-NAPPIA?

Kun painat Quest HOME-nappia:
1. Immersive VR -sessio **keskeytyy** (session.visibilityState = 'visible-blurred')
2. WebXR lopettaa kontrollerin matriisien erikoispäivityksen
3. A-Frame palaa "normaaliin" tilaan jossa getWorldPosition() toimii oikein
4. Kontrollerit näkyvät edelleen koska sessio ei ole täysin suljettu

Tämä vahvistaa että ongelma on nimenomaan WebXR immersive-vr -tilan matriisikäsittelyssä.

---

## 🧪 TESTAUSOHJE

1. Päivitä A-Frame 1.7.0
2. Vaihda `oculus-touch-controls` → `meta-touch-controls`
3. Muuta getPaddleWorldPos() käyttämään setFromMatrixPosition()
4. Testaa Quest 3:lla immersive VR -tilassa

---

## 📚 LÄHTEET

- **Quest 3 A-Frame ongelma:** https://communityforums.atmeta.com/t5/Quest-Development/Quest-3-Controllers-not-registering-with-A-Frame-in-WebXR/td-p/1192678
- **Meta Touch Controls tuki:** https://github.com/aframevr/aframe/issues/5143
- **Three.js getWorldPosition VR-bugi:** https://github.com/mrdoob/three.js/issues/23597
- **Camera world position VR:** https://github.com/mrdoob/three.js/issues/16382
- **A-Frame Quest camera issue:** https://github.com/aframevr/aframe/issues/4568
- **meta-touch-controls docs:** https://aframe.io/docs/1.7.0/components/meta-touch-controls.html

---

## ✅ YHTEENVETO

| Ongelma | Syy | Ratkaisu |
|---------|-----|----------|
| Quest 3 kontrollerit ei rekisteröidy | A-Frame 1.5.0 ei tue Quest 3:a | Päivitä 1.7.0 + meta-touch-controls |
| getWorldPosition() väärin VR:ssä | Three.js WebXR matriisi-bugi | Käytä setFromMatrixPosition() |
| Child-elementit väärässä paikassa | Parent matriisi lasketaan 2x | Älä käytä childeja tai laske itse |

**Prioriteetti:** Tee ensin A-Frame päivitys, sitten korjaa getPaddleWorldPos().
