# Airsoft Kokkola - Pelinvetovuorot

Pelinvetäjien ilmoittautumisjärjestelmä Airsoft Kokkolan peleille. Yksinkertainen kalenteripohjainen sovellus, jossa pelaajat voivat ilmoittautua pelinvetäjiksi ja varavetäjiksi.

## Ominaisuudet

### Peruskäyttäjille
- **Kalenterinäkymä** - Selkeä kuukausinäkymä tulevista peleistä
- **Ilmoittautuminen** - Ilmoittaudu päävetäjäksi tai varavetäjäksi
- **Kommentit** - Lisää kommentti ilmoittautumiseen (esim. "Tarvitaan apuvetäjä")
- **Omat varaukset** - Näe kaikki omat tulevat vetovuorot yhdessä paikassa
- **Mobiilioptimointi** - Toimii sujuvasti puhelimella

### Varavetäjäjärjestelmä
- Jos päävetäjä on jo valittu, voit ilmoittautua **varavetäjäksi**
- Jos päävetäjä peruu, ensimmäinen varavetäjä **ylennetään automaattisesti** päävetäjäksi
- Ylennetty varavetäjä saa **sähköposti-ilmoituksen**

### Ylläpitäjille (Admin)
- **Käyttäjähallinta** - Ylennä tai alenna käyttäjiä ylläpitäjiksi
- **Tilastot**:
  - Yhteensä pelatut pelit
  - Tulevat pelit
  - Aktiivisten vetäjien määrä
- **Vetäjätilastot**:
  - Vähiten pelejä viimeisen 3 kuukauden aikana
  - Graafi kaikista peleistä vetäjittäin
- **Poisto-oikeudet** - Ylläpitäjä voi poistaa kenen tahansa ilmoittautumisen

## Tekniikka

### Stack
- **Frontend**: Yksittäinen HTML-tiedosto (HTML + CSS + JavaScript)
- **Kalenteri**: FullCalendar 6.1.10
- **Tietokanta**: Supabase (PostgreSQL)
- **Autentikointi**: Supabase Auth (sähköpostivahvistus)
- **Sähköpostit**: Resend API + Supabase Edge Functions
- **Hosting**: GitHub Pages

### Tietoturva
- **Row Level Security (RLS)** - Käyttäjät voivat muokata vain omia tietojaan
- **HTTPS** - Kaikki liikenne salattu
- **Salasanojen hash**: Supabase Auth (bcrypt)
- **Sähköpostivahvistus** - Pakollinen rekisteröityessä
- **Yksityisyys**: Puhelinnumerot näkyvät vain ylläpitäjille ja itselle

## Asennus

### 1. Supabase-projekti

**Luo Supabase-projekti:**
1. Mene osoitteeseen [supabase.com](https://supabase.com)
2. Luo uusi projekti
3. Tallenna Project URL ja anon public key

**Luo tietokantataulut:**

Supabase SQL Editorissa aja:

```sql
-- Profiilit
CREATE TABLE profiles (
  id UUID REFERENCES auth.users(id) PRIMARY KEY,
  name TEXT,
  phone TEXT,
  is_admin BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Varaukset
CREATE TABLE reservations (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL,
  event_date DATE NOT NULL,
  comment TEXT,
  is_backup BOOLEAN DEFAULT FALSE,
  promoted_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  FOREIGN KEY (user_id) REFERENCES profiles(id) ON DELETE CASCADE
);

-- RLS käyttöön
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE reservations ENABLE ROW LEVEL SECURITY;

-- RLS-säännöt profiileille
CREATE POLICY "Users can view basic profile info"
  ON profiles FOR SELECT
  USING (true);

CREATE POLICY "Users can update own profile"
  ON profiles FOR UPDATE
  USING (auth.uid() = id);

CREATE POLICY "Admins can update any profile"
  ON profiles FOR UPDATE
  USING (EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND is_admin = TRUE));

-- RLS-säännöt varauksille
CREATE POLICY "Everyone can view reservations"
  ON reservations FOR SELECT
  USING (true);

CREATE POLICY "Users can insert own reservations"
  ON reservations FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can delete own reservations or admins can delete any"
  ON reservations FOR DELETE
  USING (
    auth.uid() = user_id 
    OR 
    EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND is_admin = TRUE)
  );

CREATE POLICY "Users can update own reservations"
  ON reservations FOR UPDATE
  USING (auth.uid() = user_id);

-- Tee itsesi ylläpitäjäksi (korvaa USER_ID omalla)
UPDATE profiles SET is_admin = TRUE WHERE id = 'YOUR_USER_ID';
```

**Ota sähköpostivahvistus käyttöön:**
1. Authentication → Providers → Email
2. Valitse "Confirm email" → ON
3. Authentication → URL Configuration
4. Aseta Site URL: `https://sinun-github-käyttäjä.github.io/pelinveto/`
5. Lisää Redirect URLs: `https://sinun-github-käyttäjä.github.io/pelinveto/**`

### 2. Resend (sähköpostit)

**Luo Resend-tili:**
1. Mene osoitteeseen [resend.com](https://resend.com)
2. Luo ilmainen tili
3. API Keys → Create API Key → Kopioi avain

**Luo Edge Function Supabaseen:**

1. Supabase Dashboard → Edge Functions → Create function
2. Nimi: `send-promotion-email`
3. Liitä koodi:

```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const RESEND_API_KEY = Deno.env.get('RESEND_API_KEY')
const SUPABASE_URL = Deno.env.get('SUPABASE_URL')
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')

serve(async (req) => {
  try {
    const { userId, name, eventDate } = await req.json()

    const supabaseAdmin = createClient(
      SUPABASE_URL!,
      SUPABASE_SERVICE_ROLE_KEY!
    )

    const { data: { user }, error: userError } = await supabaseAdmin.auth.admin.getUserById(userId)
    
    if (userError || !user?.email) {
      return new Response(
        JSON.stringify({ error: 'User not found or no email' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } }
      )
    }

    const res = await fetch('https://api.resend.com/emails', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${RESEND_API_KEY}`,
      },
      body: JSON.stringify({
        from: 'Pelinvetoryhmä <onboarding@resend.dev>',
        to: [user.email],
        subject: 'Olet ylennetty päävetäjäksi!',
        html: `
          <h2>Hei ${name}!</h2>
          <p>Päävetäjä perui ilmoittautumisensa, ja sinut on ylennetty <strong>päävetäjäksi</strong> päivälle <strong>${eventDate}</strong>.</p>
          <p>Tarkista lisätiedot kalenterista: <a href="https://kapistus.github.io/pelinveto/">https://kapistus.github.io/pelinveto/</a></p>
          <p>- Airsoft Kokkola</p>
        `,
      }),
    })

    const data = await res.json()
    
    return new Response(JSON.stringify(data), {
      status: res.ok ? 200 : 400,
      headers: { 'Content-Type': 'application/json' },
    })
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    )
  }
})
```

4. Edge Functions → Manage secrets → Lisää:
   - `RESEND_API_KEY` = `re_your_key_here`
5. Deploy

### 3. GitHub Pages

1. Luo uusi GitHub-repositorio (esim. "pelinveto")
2. Lataa `index.html` tiedosto repositorioon
3. Päivitä tiedostossa Supabase-tiedot:
   ```javascript
   const SUPABASE_URL = 'https://sinun-projekti.supabase.co';
   const SUPABASE_ANON_KEY = 'sinun-anon-key';
   ```
4. Settings → Pages → Source: Deploy from branch `main`
5. Sivusto on nyt osoitteessa: `https://sinun-käyttäjä.github.io/pelinveto/`

## Käyttö

### Rekisteröityminen
1. Mene sivustolle
2. Klikkaa "Tarvitsetko tunnukset? Rekisteröidy"
3. Täytä: nimi, sähköposti, puhelin, salasana
4. Tarkista sähköposti ja klikkaa vahvistuslinkkiä
5. Kirjaudu sisään

### Ilmoittautuminen vetäjäksi
1. Kirjaudu sisään
2. Klikkaa kalenterista päivää
3. Jos päivällä ei ole vetäjää:
   - Lisää kommentti (valinnainen)
   - Klikkaa "Ilmoittaudu vetäjäksi"
4. Jos päivällä on jo vetäjä:
   - Ilmoittaudu varavetäjäksi samalla tavalla

### Ilmoittautumisen peruminen
1. Klikkaa kalenterista päivää jolla olet vetäjänä
2. Klikkaa "Poista ilmoittautumiseni"
3. Vahvista
4. **Huom:** Jos olet päävetäjä ja perut, ensimmäinen varavetäjä ylennetään automaattisesti

### Ylläpito (vain admineille)
1. Klikkaa "Profiili"
2. Klikkaa "Ylläpito"-välilehti
3. Näet:
   - Käyttäjälistan (voit ylentää/alentaa admineja)
   - Tilastot
   - Vetäjätilastot graafina

## Tietokannan ylläpito

### Varmuuskopiointi
Supabase tekee automaattiset varmuuskopiot, mutta voit myös:
- Database → Table Editor → Export as CSV

### Vanhojen varausten siivous
Poista yli 6kk vanhat varaukset (valinnainen):

```sql
DELETE FROM reservations 
WHERE event_date < NOW() - INTERVAL '6 months';
```

## Vianmääritys

### "Profiilia ei löydy" -virhe rekisteröityessä
Profiilin luonti epäonnistui. Luo manuaalisesti:

```sql
INSERT INTO profiles (id, name, phone, is_admin)
VALUES ('USER_ID_FROM_AUTH_USERS', 'Nimi', '+358401234567', false);
```

### Sähköpostit menevät roskapostiin
1. Merkitse ensimmäinen sähköposti "ei roskapostia"
2. Jatkossa menee suoraan postilaatikkoon
3. **Tai** aseta oma domain Resendiin (vaatii oman verkkotunnuksen)

### Kalenterin päivitysongelmat
- Hard refresh: `Ctrl + Shift + R` (Windows/Linux) tai `Cmd + Shift + R` (Mac)
- GitHub Pages välimuisti voi kestää muutaman minuutin

### Admin-välilehti ei näy
1. Tarkista tietokannasta:
   ```sql
   SELECT id, name, is_admin FROM profiles WHERE id = 'YOUR_USER_ID';
   ```
2. Jos `is_admin` on `false`, päivitä:
   ```sql
   UPDATE profiles SET is_admin = TRUE WHERE id = 'YOUR_USER_ID';
   ```
3. Hard refresh sivusto

## Kehitysideoita tulevaisuuteen

- [ ] Toistuvat tapahtumat ("Joka lauantai")
- [ ] iCal/ICS export (puhelimen kalenteriin)
- [ ] Muistutussähköpostit 2-3 päivää ennen vetovuoroa
- [ ] Varusteiden seuranta (mitä varusteita tarvitaan)
- [ ] Läsnäolon kirjaus (kuka todella tuli paikalle)
- [ ] PWA-tuki (asenna kotinäytölle)
- [ ] Oma sähköpostidomain

## Tekijä

Projekti toteutettu Airsoft Kokkola -yhteisölle.

## Lisenssi

Vapaa käyttöön omassa porukassa. Muokkaa vapaasti tarpeisiisi.
