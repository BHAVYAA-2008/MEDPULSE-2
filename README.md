# MedPulse

Golden-hour emergency network with **three portals** and one automatic alarm:

| Portal | Who | What they do |
| --- | --- | --- |
| Patient portal | Public, patients, families, donors | Search beds, ICU, oxygen, ventilators, ice storage, antidotes and blood with live unit counts. Chat or call the hospital. Hold a bed for 15 minutes. Receive blood-request pop-ups and tick to donate. |
| Hospital portal | Hospital admin and reception | Update beds and stock with `+` / `-` and switches. Request blood. Chat with patients and donors. See incoming ambulances. |
| Ambulance portal | Drivers and 108 dispatchers | Pick a hospital, start a trip, live GPS and ETA. |
| (no portal) Traffic police | Control rooms | Receive an **automatic alarm** (WhatsApp, SMS, optional webhook) when a trip starts. They clear junctions on their own. |

Stack: React + Vite PWA, Supabase (Postgres, Auth, Realtime, Edge Functions), Leaflet + OpenStreetMap, OSRM routing, WhatsApp Cloud API and Fast2SMS for messages.

---

## Fastest path to a public link (about 15 minutes)

1. **Upload to GitHub.** Create an empty repository, then upload every file and folder from this project (drag and drop works on github.com, or use the git commands at the bottom).
2. **Import it in Vercel** (vercel.com, "Add New Project", pick the repository). Framework: Vite. Build command `npm run build`, output `dist`. Deploy.
3. You instantly get a link like `https://medpulse-xxxx.vercel.app`.
   * `/demo.html` is a self-contained **working demo** (no backend needed).
   * The real app (`/`) shows a "Backend not connected" notice until you finish the Supabase steps below.

## Connect the real backend

### 1. Supabase project
1. Create a project at supabase.com, region **Mumbai (ap-south-1)**.
2. In **SQL Editor**, run these files in order: `supabase/migrations/20260101000001_schema.sql`, `...02_rls.sql`, `...03_functions.sql`, `...04_realtime_cron.sql`, then `supabase/seed.sql`.
   (Or with the CLI: `supabase link --project-ref YOUR_REF && supabase db push`.)
3. **Authentication > Providers > Phone**: enable it. For a quick prototype use *Test phone numbers and OTPs* (add your own number with a fixed code) so no SMS provider is needed. For real SMS, connect Twilio or MSG91 here.
4. Copy **Project URL** and **anon key** (Settings > API).

### 2. Frontend environment
Local: `cp .env.example .env`, fill in the two values, then `npm install` and `npm run dev`.
Vercel: Project Settings > Environment Variables, add `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY`, then redeploy.

### 3. Create your roles
Everyone starts as a normal user. Log in once with your phone number, then in the SQL Editor (phone is stored as digits with country code, check the `profiles` table):

```sql
-- hospital admin for a seeded hospital
update profiles set role = 'hospital_admin' where phone = '91XXXXXXXXXX';
insert into hospital_staff (hospital_id, user_id)
select h.id, p.id from hospitals h, profiles p
where h.name = 'Apollo Greams Road' and p.phone = '91XXXXXXXXXX';

-- ambulance driver
update profiles set role = 'driver' where phone = '91YYYYYYYYYY';
```

Real hospitals are added by you in the `hospitals` table (`approved = true` makes them public). This is your verification step.

### 4. Edge Functions (blood alerts and police alarm)
```bash
npm i -g supabase
supabase login
supabase link --project-ref YOUR_REF
supabase functions deploy send-blood-alert
supabase functions deploy start-trip-alarm
supabase secrets set APP_URL=https://your-app.vercel.app
```
Optional secrets (functions work in dry-run mode without them, donors still get the in-app pop-up):

| Secret | Purpose |
| --- | --- |
| `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_ID` | WhatsApp Cloud API credentials |
| `WA_TEMPLATE_BLOOD`, `WA_TEMPLATE_POLICE` | Approved template names (defaults `blood_request`, `police_alarm`) |
| `FAST2SMS_KEY`, `FAST2SMS_SENDER` | Fast2SMS (needs DLT registration) |
| `SMS_TEMPLATE_BLOOD`, `SMS_TEMPLATE_POLICE` | DLT template IDs |
| `ALARM_WEBHOOK_URL` | Forward every police alarm to your own system |
| `OSRM_URL` | Your own routing server (the public demo server is for light testing only) |

Template body for `blood_request`: `Urgent: {{1}} needs {{2}} blood, {{3}} unit(s). Open MedPulse to respond.`
Template body for `police_alarm`: `Ambulance alert ({{1}}) heading to {{2}}, ETA {{3}} min. Live track: {{4}}`

## How to test end to end
1. Two browsers (or one normal and one private window). Browser A: log in as the hospital admin. Browser B: log in as another user, open **My donor settings**, set blood group `AB-`, tick the consent box, press **Use my current location**.
2. Browser A: Hospital portal > Request blood `AB-`. Browser B shows the pop-up within about a second. A user with another blood group sees nothing.
3. Browser B ticks the box and presses **Contact hospital**. The chat opens, and the donor appears in Browser A under *Donors who can come*.
4. Browser A: tap `+` on ICU beds. The Patient portal updates live.
5. Patient portal: hold a bed. Count drops by one, and returns after 15 minutes (pg_cron) or on Release.
6. Driver account: Ambulance portal > Start trip. The hospital sees the incoming ambulance, the alarm result appears, and the family link opens the live map without login.

## Project structure
```
src/                 React app (pages for the three portals, shared components, lib helpers)
public/              PWA manifest, service worker, icon, demo.html (standalone demo)
supabase/migrations  Schema, Row Level Security, functions, realtime and cron
supabase/functions   Edge Functions: send-blood-alert, start-trip-alarm
supabase/seed.sql    Sample Chennai hospitals and a placeholder police zone
docs/                Architecture notes and launch checklist
```

## Important notes before real users
* This is a **prototype**. It has not been load tested or security audited. Read `docs/LAUNCH_CHECKLIST.md`.
* Replace every placeholder phone number in `seed.sql`. Never alarm a number you do not own while testing.
* OpenStreetMap tiles and the public OSRM server have fair-use limits. Use a paid tile provider and your own OSRM for launch.
* A browser tab can be paused by the phone when the screen is off. The app asks for a screen wake lock; for reliable background GPS wrap the app with Capacitor later.
* MedPulse supports the 108 helpline and doctors. It does not replace them. Show that clearly to users.
* Handle personal data under India's DPDP Act 2023: privacy policy, consent text and a way to delete accounts.

## Push to GitHub with git
```bash
git init
git add .
git commit -m "MedPulse prototype"
git branch -M main
git remote add origin https://github.com/YOUR-USER/medpulse.git
git push -u origin main
```

## License
MIT
