# Cloud Supabase Setup: Produktinfo-Portal

## 1. Supabase Projekt Setup

### Cloud Projekt erstellen

1. Auf [Supabase.com](https://supabase.com):
   - Neues Projekt anlegen
   - Region: EU Central (Frankfurt) wählen
   - Datenbankpasswort sicher speichern
   - Organization und Projekt Namen festlegen

2. Credentials sichern:
   - Project URL
   - anon/public key
   - service_role key (für Admin-Operationen)
   - Database Connection String

### Umgebungsvariablen vorbereiten (.env)
```env
NEXT_PUBLIC_SUPABASE_URL=https://[YOUR-PROJECT].supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJh...
SUPABASE_SERVICE_ROLE_KEY=eyJh...
DATABASE_URL=postgresql://postgres:[YOUR-PASSWORD]@db.[YOUR-PROJECT].supabase.co:5432/postgres
```

## 2. Datenbank-Schema Implementation

1. Schema importieren:
   - SQL Editor in Supabase Dashboard öffnen
   - Inhalt von supabase-schema.sql einfügen und ausführen
   - Auf erfolgreiche Ausführung prüfen

2. Storage-Buckets erstellen:
```sql
-- In der Supabase SQL Editor
INSERT INTO storage.buckets (id, name, public)
VALUES 
  ('product-images', 'product-images', false),  -- Nicht public, da Intranet
  ('product-manuals', 'product-manuals', false);

-- Storage Policies für authentifizierte Benutzer
CREATE POLICY "Authenticated users can view files"
  ON storage.objects FOR SELECT
  TO authenticated
  USING (bucket_id IN ('product-images', 'product-manuals'));
```

## 3. Grundkonfiguration

### Storage Konfiguration

1. Bucket-Einstellungen für Produktbilder:
```sql
UPDATE storage.buckets
SET 
  file_size_limit = 2097152,
  allowed_mime_types = ARRAY['image/jpeg', 'image/png']
WHERE id = 'product-images';
```

2. Bucket-Einstellungen für Anleitungen:
```sql
UPDATE storage.buckets
SET 
  file_size_limit = 10485760,
  allowed_mime_types = ARRAY['application/pdf']
WHERE id = 'product-manuals';
```

### Initiale Admin-Benutzer

1. Admin Rolle erstellen:
```sql
CREATE TYPE user_role AS ENUM ('admin', 'product_manager');

ALTER TABLE auth.users
ADD COLUMN role user_role DEFAULT 'product_manager';
```

2. Admin-Benutzer über Supabase Dashboard:
   - Authentication > Users
   - "Invite user" für initiale Admins
   - SQL ausführen für Admin-Rechte:
```sql
UPDATE auth.users
SET role = 'admin'
WHERE email = 'admin@yourcompany.com';
```

## 4. Basis-Tests

### 1. Test-Daten einfügen
```sql
-- Test-Produkt
INSERT INTO public.products (
    sku,
    name,
    description
) VALUES (
    '12345',
    'Test Produkt',
    'Beschreibung für Testprodukt'
);

-- Test-Charge
INSERT INTO public.product_batches (
    product_id,
    po_number
) VALUES (
    '12345',
    'PO-2025-001'
);
```

### 2. Vererbungslogik testen
```sql
-- Zweite Charge mit Vererbung
INSERT INTO public.product_batches (
    product_id,
    po_number,
    inherit_previous,
    inherit_from_batch_id
) VALUES (
    '12345',
    'PO-2025-002',
    true,
    1
);

-- Test-Query für Vererbung
SELECT 
    b.*,
    COALESCE(b.override_name, p.name) as effective_name,
    COALESCE(b.override_description, p.description) as effective_description
FROM product_batches b
JOIN products p ON b.product_id = p.sku
WHERE b.product_id = '12345';
```

## 5. Admin Client Setup

### Next.js Projekt initialisieren
```bash
npx create-next-app@latest admin-client --typescript --tailwind --app
cd admin-client
npm install @supabase/supabase-js @tanstack/react-query @hookform/resolvers/zod zod
```

### Supabase Client Konfiguration
```typescript
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)
```

## 6. Nächste Schritte

1. Admin Client entwickeln:
   - Auth Flow implementieren
   - Produkt CRUD Operationen
   - Chargen-Management
   - File Upload/Download

2. Testing:
   - Auth Flow
   - RLS Policies
   - Vererbungssystem
   - File Management

3. Monitoring einrichten:
   - Database Logs aktivieren
   - Performance Monitoring
   - Error Tracking