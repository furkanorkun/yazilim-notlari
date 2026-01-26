# React useContext Hook'u

React **useContext**, bileşenler arasında veri taşırken "Prop Drilling" sorununu çözmek için geliştirilmiş güçlü bir Hook'tur. Özellikle global veriler (Tema, Kullanıcı Oturumu, Dil Seçimi vb.) için idealdir.

## Temel Mantık: Prop Drilling vs Context

Normalde veri, ebeveynden çocuğa **props** yoluyla iletilir. Ancak veri çok derindeki bir bileşene lazım olduğunda, aradaki tüm bileşenlerin bu veriyi props olarak taşıması gerekir. Context, veriyi ağacın en tepesinden yayınlar (broadcast) ve ihtiyacı olan bileşen ne kadar derinde olursa olsun bu veriyi çeker.

### Prop Drilling Örneği
```
App
├── Header (props ile veri taşır)
    ├── Nav (props ile veri taşır)
        └── Button (sonunda veriyi kullanır)
```

### Context Çözümü
```
App (Provider ile veri yayınlar)
├── Header
    ├── Nav
        └── Button (doğrudan veriyi çeker)
```

## Kurulum ve Kullanım

Standart kullanım 3 aşamada oluşur: **Oluşturma (Create)**, **Sağlama (Provide)** ve **Tüketme (Consume)**.

### 1. Context Oluşturma
Genellikle ayrı bir dosyada oluşturulur.

```javascript
// ThemeContext.js
import { createContext } from 'react';

// Başlangıç değeri verebilirsin (opsiyonel ama önerilir)
const ThemeContext = createContext('light');

export default ThemeContext;
```

### 2. Provider ile Sarmalama
Veriyi kullanacak bileşenlerin en tepesine Provider yerleştirilir.

```javascript
// App.js
import ThemeContext from './ThemeContext';

function App() {
  const themeValue = 'dark';

  return (
    // value prop'u ile veriyi ağaca enjekte ediyoruz
    <ThemeContext.Provider value={themeValue}>
      <Header />
      <MainContent />
    </ThemeContext.Provider>
  );
}
```

### 3. Veriyi Kullanma
Artık MainContent içindeki herhangi bir bileşen bu veriye erişebilir.

```javascript
// Button.js
import { useContext } from 'react';
import ThemeContext from './ThemeContext';

function Button() {
  // Hook'u kullanarak değeri alıyoruz
  const theme = useContext(ThemeContext);

  return <button className={theme}>Temalı Buton</button>;
}
```

## Best Practices

Profesyonel projelerde Context'i ham haliyle kullanmak yerine aşağıdaki pattern'leri uygularız.

### Kural 1: Custom Hook ile Kapsülleme
`useContext(ThemeContext)`'i her dosyada çağırmak yerine, özel bir hook yazarak Context mantığını soyutlayın. Bu, Context'in Provider dışında kullanılıp kullanılmadığını kontrol etmenizi sağlar.

```javascript
// hooks/useTheme.js
import { useContext } from "react";
import { ThemeContext } from "../context/ThemeContext";

export const useTheme = () => {
  const context = useContext(ThemeContext);

  if (context === undefined) {
    throw new Error("useTheme, ThemeProvider içerisinde kullanılmalıdır!");
  }

  return context;
};
```

### Kural 2: Logic ve State'i Provider İçine Gizleme
Context dosyasını sadece bir "veri taşıyıcı" değil, o alanla ilgili mantığın (business logic) merkezi yapın.

```javascript
// context/AuthContext.js
import { createContext, useState, useContext } from "react";

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);

  const login = (userData) => {
    // API isteği vb. işlemleri burada yapabilirsin
    setUser(userData);
  };

  const logout = () => {
    setUser(null);
  };

  // Value objesi hem state'i hem fonksiyonları içerir
  const value = { user, login, logout };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};

// Custom Hook export ediyoruz
export const useAuth = () => useContext(AuthContext);
```

### Kural 3: Performans İçin UseMemo Kullanımı
Context value prop'u her değiştiğinde, o context'i tüketen tüm bileşenler yeniden render (re-render) olur. Eğer value içine her render'da yeni referansı olan bir obje `{ user, login }` verirseniz, gereksiz renderlara sebep olursunuz.

**Çözüm:** value objesini `useMemo` ile sarmalayın.

```javascript
// AuthContext.js içinde
import { useMemo } from 'react';

const value = useMemo(() => ({
  user,
  login,
  logout
}), [user]); // Sadece user değiştiğinde value referansı yenilenir.
```

### Kural 4: Context Splitting (Bağlam Bölme)
Büyük bir state objeniz varsa ve bir bileşen sadece veriyi okuyor, diğeri ise sadece güncelliyorsa; bunları tek Context'e koymak performans sorunu yaratabilir.

**Senaryo:** SettingsContext içinde hem settings verisi hem de updateSettings fonksiyonu var. Sadece updateSettings kullanan bir buton, settings verisi değiştiğinde (kendisi veriyi kullanmasa bile) re-render olur.

**Çözüm:** State ve Dispatch (Fonksiyon) contextlerini ayırın.

- SettingsStateContext (Sadece veri)
- SettingsDispatchContext (Sadece güncelleme fonksiyonları)

Bu desen genellikle `useReducer` ile birlikte kullanılır.

## Ne Zaman useContext Kullanmamalısın?

Context güçlüdür ama her durumda kullanılmamalıdır. Aşağıdaki durumlarda dikkatli olun:

- **Sık değişen veriler:** Eğer veri saniyede değişiyorsa (örneğin mouse pozisyonu, timer, animasyon state'i) Context performans sorunlarına yol açabilir. Bu durumlar için Redux, Zustand gibi state yönetim kütüphaneleri daha uygundur.
- **Sadece 1-2 seviye derinlik:** Eğer veri sadece 1-2 seviye derinlikteki bileşenler arasında paylaşılıyorsa, prop drilling yapmak daha basit ve izlenebilir olabilir.
- **Bileşen Kompozisyonu:** Bazen Context yerine "Component Composition" (Bileşeni prop olarak geçmek) sorunu daha temiz çözer.

## Özet Tablo

| Özellik | Açıklama |
|---------|----------|
| **Provider** | Veriyi ağaca enjekte eden sarmalayıcı bileşen |
| **Consumer** | `useContext` hook'u ile veriyi çeken bileşen |
| **Kritik Nokta** | Provider'ın `value` prop'u değişirse, tüm tüketiciler re-render olur |
| **Best Practice** | `useMemo` ile value'yu önbellekle, Custom Hook yaz |

> **İpucu:** Context'i aşırı kullanmayın. Küçük projelerde basit prop passing yeterli olabilir. Büyük projelerde ise daha gelişmiş state management çözümlerini değerlendirin.

## useContext ve useReducer Kombinasyonu

Senaryomuz bir "Kullanıcı Yetkilendirme (Auth)" sistemi olsun. Bu senaryo useState ile yönetilmeye kalkıldığında isLoading, error, user, permissions gibi state'ler birbirine girmeye başlar. useReducer state yönetimini disipline etmek için tercih edilir.

### 1. Dosya Yapısı ve Mantık

```javascript
// contexts/AuthContext.js
import { createContext, useContext, useReducer, useMemo } from "react";

// 1. BAŞLANGIÇ STATE'İ (Initial State)
const initialState = {
  user: null,          // Kullanıcı objesi (id, name, role vs.)
  isAuthenticated: false,
  isLoading: false,
  error: null,
};

// 2. REDUCER (State'in nasıl değişeceğinin kuralları)
// Reducer, state'i doğrudan değiştirmez, yeni bir kopyasını döner.
const authReducer = (state, action) => {
  switch (action.type) {
    case "LOGIN_START":
      return { ...state, isLoading: true, error: null };
    
    case "LOGIN_SUCCESS":
      return { 
        ...state, 
        isLoading: false, 
        isAuthenticated: true, 
        user: action.payload 
      };
    
    case "LOGIN_FAILURE":
      return { 
        ...state, 
        isLoading: false, 
        isAuthenticated: false, 
        error: action.payload 
      };
    
    case "LOGOUT":
      return { ...initialState }; // State'i sıfırlar

    default:
      throw new Error(`Bilinmeyen action tipi: ${action.type}`);
  }
};

// 3. CONTEXT OLUŞTURMA
const AuthContext = createContext();

// 4. PROVIDER BİLEŞENİ
export const AuthProvider = ({ children }) => {
  const [state, dispatch] = useReducer(authReducer, initialState);

  // Helper Fonksiyonlar (Action Creators gibi davranır)
  // Bu fonksiyonlar UI içinde dispatch çağırmak yerine temiz bir arayüz sunar.
  const login = async (username, password) => {
    dispatch({ type: "LOGIN_START" });

    try {
      // Simüle edilmiş API isteği
      await new Promise((resolve) => setTimeout(resolve, 1000));

      if (username === "admin" && password === "1234") {
        const fakeUser = { id: 1, name: "Admin User", role: "admin" };
        dispatch({ type: "LOGIN_SUCCESS", payload: fakeUser });
      } else {
        throw new Error("Kullanıcı adı veya şifre hatalı!");
      }
    } catch (err) {
      dispatch({ type: "LOGIN_FAILURE", payload: err.message });
    }
  };

  const logout = () => {
    dispatch({ type: "LOGOUT" });
  };

  // 5. PERFORMANS OPTİMİZASYONU (useMemo)
  // Value objesi her render'da yeniden oluşmasın diye useMemo kullanıyoruz.
  // Sadece state değiştiğinde referans yenilenir.
  const value = useMemo(() => ({
    ...state, // user, isAuthenticated, isLoading, error
    login,
    logout
  }), [state]); 

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};

// 6. CUSTOM HOOK
export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth, AuthProvider içerisinde kullanılmalıdır!");
  }
  return context;
};
```

### 2. Uygulama İçerisinde Kullanımı

```javascript
// App.js
import { AuthProvider } from "./contexts/AuthContext";
import Dashboard from "./components/Dashboard";
import LoginForm from "./components/LoginForm";
import { useAuth } from "./contexts/AuthContext";

// İçerik Bileşeni (Auth durumuna göre ekran değişir)
const MainContent = () => {
  const { isAuthenticated } = useAuth(); // Hook'u kullanıyoruz

  return isAuthenticated ? <Dashboard /> : <LoginForm />;
};

function App() {
  return (
    <AuthProvider>
      <div className="app-container">
        <h1>SaaS Yönetim Paneli</h1>
        <MainContent />
      </div>
    </AuthProvider>
  );
}

export default App;
```

### 3. Bileşenlerde Veri Kullanımı

```javascript
// components/LoginForm.js
import { useState } from "react";
import { useAuth } from "../contexts/AuthContext";

export default function LoginForm() {
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  
  // Context'ten ihtiyacımız olanları çekiyoruz
  const { login, isLoading, error } = useAuth();

  const handleSubmit = (e) => {
    e.preventDefault();
    login(username, password); // Logic context içinde gizli!
  };

  return (
    <form onSubmit={handleSubmit} style={{ border: "1px solid #ccc", padding: "20px" }}>
      <h3>Giriş Yap</h3>
      {error && <p style={{ color: "red" }}>{error}</p>}
      
      <div>
        <label>Kullanıcı Adı (admin): </label>
        <input value={username} onChange={(e) => setUsername(e.target.value)} />
      </div>
      <br />
      <div>
        <label>Şifre (1234): </label>
        <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
      </div>
      <br />
      <button type="submit" disabled={isLoading}>
        {isLoading ? "Giriş Yapılıyor..." : "Giriş"}
      </button>
    </form>
  );
}
```

```javascript
// components/Dashboard.js
import { useAuth } from "../contexts/AuthContext";

export default function Dashboard() {
  const { user, logout } = useAuth();

  return (
    <div style={{ background: "#e0f7fa", padding: "20px" }}>
      <h2>Hoşgeldin, {user.name}</h2>
      <p>Yetki Seviyesi: <strong>{user.role}</strong></p>
      
      {user.role === 'admin' && (
        <div className="admin-panel">
          🚧 Sadece adminlerin görebileceği ayarlar buraya...
        </div>
      )}

      <button onClick={logout} style={{ marginTop: "10px" }}>
        Çıkış Yap
      </button>
    </div>
  );
}
```

### Neden Bu Yapı?

- **State Logic Ayrımı:** LoginForm içinde fetch işlemleri veya try-catch blokları yok. UI sadece "görüntüleme" ve "tetikleme" işini yapıyor. Kirli işler AuthContext içinde.
- **Öngörülebilirlik:** useReducer sayesinde state'in LOGIN_START olmadan LOGIN_SUCCESS olamayacağını biliyoruz. State geçişleri net.
- **Performans:** useMemo sayesinde, sadece login fonksiyonunu kullanan bir bileşen, user değiştiğinde gereksiz render olabilir ancak bunu React.memo veya context'i bölerek (DispatchContext / StateContext) daha da optimize edebiliriz. Bu yapı çoğu durum için yeterince performanslıdır.
