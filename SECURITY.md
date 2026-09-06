# API key और "छिपाने" की सच्चाई

---

## पहले एक ज़रूरी बात — Firebase की web API key छिपाई नहीं जा सकती

आपने कहा "API hide करो"। मैं यह कर *सकता* था:

```js
// base64, या टुकड़ों में जोड़कर, या rot13 करके...
var k = atob('QUl6YVN5QTRrSW56...');
```

**पर यह सुरक्षा नहीं, दिखावा है।** browser को असली key चाहिए ही — वरना
Firebase से बात नहीं होगी। कोई भी DevTools → Network खोलकर पहली request
में उसे साफ़ पढ़ लेगा। मैंने जानबूझकर यह नहीं किया, क्योंकि इससे आपको
सुरक्षित होने का *भ्रम* होता और असली बचाव पीछे छूट जाता।

Google खुद कहता है यह key **secret नहीं** है — यह सिर्फ़ पहचान बताती है कि
request किस project की है, अधिकार नहीं देती।

> **असली सवाल यह नहीं कि "key कौन देख सकता है"**,
> **बल्कि यह कि "key हाथ लग जाने पर कोई कर क्या सकता है"।**

नीचे उसी का जवाब है।

---

## key हाथ लगने पर हमलावर क्या कर सकता है — और क्या रोकता है

| वो क्या करेगा | उसे कौन रोकता है | अभी हालत |
|---|---|---|
| सीधे Firestore से data पढ़ना | `firestore.rules` | ✅ 77 tests पास |
| फ़र्ज़ी order/withdrawal बनाना | rules + Cloud Functions | ✅ पैसा पूरा server पर |
| curl/Postman से Functions बुलाना | App Check | ⚠️ आपको चालू करना है |
| अपनी website पर आपका Firebase चलाना | API key restriction | ⚠️ आपको लगाना है |
| दूसरे domain से login कराना | Authorized domains | ⚠️ जाँच लें |
| RTDB से सब उठा लेना | `database.rules.json` | ✅ अभी बंद किया |

तीन ⚠️ आपके Console में करने हैं — code से नहीं होते। नीचे तरीक़ा है।

---

## 1. API key restriction (सबसे ज़रूरी, 5 मिनट)

यह key को *बेकार* कर देता है अगर वो आपकी website के बाहर इस्तेमाल हो।

1. [Google Cloud Console → Credentials](https://console.cloud.google.com/apis/credentials)
   खोलें, project `sewaastra` चुनें
2. तीनों "Browser key (auto created by Firebase)" एक-एक करके खोलें
3. **Application restrictions** → `Websites` चुनें
4. सिर्फ़ अपने domain डालें:

```
https://sewaastra.web.app/*
https://sewaastra.firebaseapp.com/*
https://आपका-domain.com/*
http://localhost:*        ← सिर्फ़ testing के दौरान, बाद में हटा दें
```

5. **API restrictions** → `Restrict key` → सिर्फ़ ये चुनें:
   - Identity Toolkit API
   - Token Service API
   - Cloud Firestore API
   - Firebase Installations API
   - Cloud Functions API
   - Firebase App Check API

अब वही key किसी और website पर डाली जाए तो Google खुद मना कर देगा।
**यही असली "hide" है** — key दिखती रहे, पर काम न करे।

---

## 2. App Check चालू करें

curl, Postman, script — कोई भी आपकी Functions और Firestore को सीधे
नहीं बुला पाएगा।

1. Firebase Console → **App Check**
2. तीनों web apps register करें → provider **reCAPTCHA Enterprise** (या v3)
3. site key लेकर हर app के `<head>` में डालें:

```html
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-check-compat.js"></script>
<script>
  firebase.appCheck().activate('आपकी-recaptcha-site-key', true);
</script>
```

4. पहले **Monitor** mode में 1 हफ़्ता चलाएँ (कुछ block नहीं होता, सिर्फ़ गिनती दिखती है)
5. सब ठीक लगे तो **Enforce** करें
6. फिर deploy से `ENFORCE_APP_CHECK=false` हटा दें

> ⚠️ Enforce करने से पहले `ENFORCE_APP_CHECK=false` हटाना **मत** भूलें —
> वरना Functions खुली रह जाएँगी।

---

## 3. Authorized domains

Firebase Console → Authentication → Settings → **Authorized domains**

सिर्फ़ अपने domain रखें। `localhost` production में हटा दें —
वरना कोई अपने कंप्यूटर पर आपका login page चलाकर असली users का OTP माँग सकता है।

---

## इस बार code में क्या ठीक किया

### 🔴 Realtime Database खुला पड़ा था

तीनों apps के config में `databaseURL` था, पर:
- कोई app RTDB इस्तेमाल **नहीं** करती (0 calls)
- उसके **कोई rules नहीं** थे

अगर project में RTDB instance बना है, तो public key से कोई भी उसे पूरा
पढ़/लिख सकता था — Firestore की सारी मेहनत के बग़ल से निकलकर।

दो काम किए:
- `database.rules.json` बनाया — सब बंद (`.read: false`, `.write: false`)
- तीनों apps से `databaseURL` हटाया (जो इस्तेमाल ही नहीं होता, उसका पता देने की ज़रूरत नहीं)

### 🔴 CSP ग्राहकों का OTP login तोड़ रही थी

v2 में मैंने CSP लगाई थी, पर `frame-src` में `https://www.google.com`
छूट गया था। Customer app phone OTP के लिए `RecaptchaVerifier` इस्तेमाल
करती है, और reCAPTCHA का iframe वहीं से आता है।

**यानी CSP deploy होते ही किसी ग्राहक का OTP login नहीं चलता।** अब जोड़ दिया।

### 🟡 CSP और सख़्त की

| बदलाव | फ़ायदा |
|---|---|
| `connect-src` सीमित | **XSS हो भी जाए तो चुराया data हमलावर के server पर नहीं जा सकता** |
| `base-uri 'self'` → `'none'` | `<base>` से सारे relative URL hijack करना बंद |
| `frame-ancestors 'none'` | clickjacking बंद |
| `object-src 'none'` | plugin injection बंद |
| `X-Frame-Options: SAMEORIGIN` → `DENY` | और सख़्त |
| असली CSP **header** (firebase.json में) | `frame-ancestors` meta tag से लागू नहीं होता — header से होता है |

`connect-src` वाली लाइन सबसे कीमती है। XSS की असली चोट data बाहर भेजने
में है — अब वो सिर्फ़ Google/Firebase और OpenStreetMap तक जा सकता है।

जाँचा भी: तीनों apps का **कोई भी बाहरी resource block नहीं होगा**।

---

## करने की सूची

- [ ] API key restriction — Cloud Console (5 मिनट, सबसे ज़्यादा फ़ायदा)
- [ ] `firebase deploy --only database` — RTDB बंद करने के लिए
- [ ] App Check — Monitor → 1 हफ़्ता → Enforce
- [ ] Authorized domains से `localhost` हटाएँ
- [ ] `ENFORCE_APP_CHECK=false` हटाएँ (App Check चालू होने के बाद)

पहला काम सबसे ज़रूरी है। बाक़ी सब हो चुका हो और key खुली रहे, तो कोई भी
आपके project पर अपनी app चला सकता है — और आपका bill भर सकता है।
