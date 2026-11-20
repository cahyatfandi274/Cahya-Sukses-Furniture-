# Cahya-Sukses-Furniture-
-- ============================================================================
-- CAHYA SUKSES FURNITURE SMART CALCULATOR - SUPABASE POSTGRESQL MIGRATION
-- ============================================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. users (extends auth.users)
CREATE TABLE public.users (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('Admin', 'Estimator', 'Staff')) DEFAULT 'Estimator',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. customers
CREATE TABLE public.customers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  phone TEXT,
  address TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. projects
CREATE TABLE public.projects (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_code TEXT UNIQUE NOT NULL,
  customer_id UUID REFERENCES public.customers(id) ON DELETE SET NULL,
  title TEXT NOT NULL,
  project_date DATE NOT NULL DEFAULT CURRENT_DATE,
  status TEXT NOT NULL DEFAULT 'Draft' CHECK (status IN ('Draft','Penawaran','Menunggu Persetujuan','Disetujui','Produksi','Instalasi','Selesai','Dibatalkan')),
  total_hpp NUMERIC(15,2) DEFAULT 0,
  offer_price NUMERIC(15,2) DEFAULT 0,
  notes TEXT,
  created_by UUID REFERENCES public.users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 4. furniture_types, 5. furniture_templates, 6. component_templates
CREATE TABLE public.furniture_types (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  description TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.furniture_templates (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  furniture_type_id UUID REFERENCES public.furniture_types(id) ON DELETE CASCADE,
  template_name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.component_templates (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  template_id UUID REFERENCES public.furniture_templates(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  default_length_cm NUMERIC(10,2) NOT NULL,
  default_width_cm NUMERIC(10,2) NOT NULL,
  default_thickness_mm NUMERIC(6,2) NOT NULL,
  default_qty INTEGER NOT NULL DEFAULT 1,
  grain_direction TEXT DEFAULT 'any',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 7. materials, 8. material_sizes, 9. material_prices (Price History Snapshot)
CREATE TABLE public.materials (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  code TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  category TEXT NOT NULL,
  specification TEXT,
  thickness_mm NUMERIC(6,2) DEFAULT 0,
  purchase_unit TEXT NOT NULL,
  current_price NUMERIC(15,2) NOT NULL DEFAULT 0,
  supplier TEXT,
  default_waste_pct NUMERIC(5,2) DEFAULT 10,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.material_sizes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  material_id UUID REFERENCES public.materials(id) ON DELETE CASCADE,
  length_cm NUMERIC(10,2) NOT NULL,
  width_cm NUMERIC(10,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.material_prices (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  material_id UUID REFERENCES public.materials(id) ON DELETE CASCADE,
  old_price NUMERIC(15,2) NOT NULL,
  new_price NUMERIC(15,2) NOT NULL,
  change_note TEXT,
  changed_by UUID REFERENCES public.users(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 10. accessories, 11. finishing_items, 12. labor_rates
CREATE TABLE public.accessories (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  material_id UUID REFERENCES public.materials(id) ON DELETE CASCADE,
  default_per_door NUMERIC(6,2) DEFAULT 2,
  default_per_drawer NUMERIC(6,2) DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.finishing_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  material_id UUID REFERENCES public.materials(id),
  consumption_per_m2 NUMERIC(10,4) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.labor_rates (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  division TEXT NOT NULL,
  rate_type TEXT CHECK (rate_type IN ('daily','borongan')),
  default_rate NUMERIC(15,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 13. project_items, 14. item_components, 15. material_usage, 16. calculations, 17. operational_costs
CREATE TABLE public.project_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID REFERENCES public.projects(id) ON DELETE CASCADE,
  furniture_type_id UUID REFERENCES public.furniture_types(id),
  item_name TEXT NOT NULL,
  unit_qty INTEGER NOT NULL DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.item_components (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_item_id UUID REFERENCES public.project_items(id) ON DELETE CASCADE,
  component_name TEXT NOT NULL,
  length_cm NUMERIC(10,2) NOT NULL,
  width_cm NUMERIC(10,2) NOT NULL,
  thickness_mm NUMERIC(6,2) NOT NULL,
  qty INTEGER NOT NULL,
  material_id UUID REFERENCES public.materials(id),
  snapshot_unit_price NUMERIC(15,2) NOT NULL,
  hpl_sides INTEGER DEFAULT 0,
  edging_sides INTEGER DEFAULT 0,
  grain_direction TEXT DEFAULT 'any',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.material_usage (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  calculation_id UUID NOT NULL,
  material_id UUID REFERENCES public.materials(id),
  net_area_m2 NUMERIC(12,4),
  purchased_qty NUMERIC(12,2),
  offcut_area_m2 NUMERIC(12,4),
  snapshot_price NUMERIC(15,2),
  total_cost NUMERIC(15,2),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.calculations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID REFERENCES public.projects(id) ON DELETE CASCADE,
  version_number INTEGER NOT NULL DEFAULT 1,
  total_material_cost NUMERIC(15,2) NOT NULL,
  total_finishing_cost NUMERIC(15,2) NOT NULL,
  total_labor_cost NUMERIC(15,2) NOT NULL,
  total_operational_cost NUMERIC(15,2) NOT NULL,
  total_hpp NUMERIC(15,2) NOT NULL,
  pricing_method TEXT CHECK (pricing_method IN ('markup','margin')),
  target_percentage NUMERIC(6,2) NOT NULL,
  final_offer_price NUMERIC(15,2) NOT NULL,
  snapshot_payload JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.operational_costs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  calculation_id UUID REFERENCES public.calculations(id) ON DELETE CASCADE,
  cost_name TEXT NOT NULL,
  cost_type TEXT CHECK (cost_type IN ('fixed','percent')),
  input_value NUMERIC(15,2) NOT NULL,
  calculated_amount NUMERIC(15,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 18. quotations, 19. quotation_items, 20. expenses, 21. project_payments, 22. settings
CREATE TABLE public.quotations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID REFERENCES public.projects(id) ON DELETE CASCADE,
  quotation_number TEXT UNIQUE NOT NULL,
  subtotal NUMERIC(15,2) NOT NULL,
  discount NUMERIC(15,2) DEFAULT 0,
  tax_amount NUMERIC(15,2) DEFAULT 0,
  grand_total NUMERIC(15,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.quotation_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  quotation_id UUID REFERENCES public.quotations(id) ON DELETE CASCADE,
  description TEXT NOT NULL,
  volume NUMERIC(10,2) NOT NULL,
  unit TEXT NOT NULL,
  unit_price NUMERIC(15,2) NOT NULL,
  amount NUMERIC(15,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.expenses (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID REFERENCES public.projects(id) ON DELETE CASCADE,
  expense_date DATE NOT NULL DEFAULT CURRENT_DATE,
  category TEXT NOT NULL,
  description TEXT,
  amount NUMERIC(15,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.project_payments (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID REFERENCES public.projects(id) ON DELETE CASCADE,
  payment_date DATE NOT NULL DEFAULT CURRENT_DATE,
  payment_type TEXT NOT NULL,
  amount NUMERIC(15,2) NOT NULL,
  payment_method TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.settings (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  company_name TEXT DEFAULT 'Cahya Sukses Furniture',
  default_saw_kerf_mm NUMERIC(5,2) DEFAULT 3.0,
  default_tax_pct NUMERIC(5,2) DEFAULT 0,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Row Level Security (RLS)
ALTER TABLE public.projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.materials ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.calculations ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Authenticated users full access" ON public.projects FOR ALL USING (auth.role() = 'authenticated');
CREATE POLICY "Authenticated read materials" ON public.materials FOR SELECT USING (auth.role() = 'authenticated');
CREATE POLICY "Admin & Estimator manage materials" ON public.materials FOR ALL USING (
  EXISTS (SELECT 1 FROM public.users u WHERE u.id = auth.uid() AND u.role IN ('Admin','Estimator'))
);
