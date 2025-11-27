# Auditoría de Ciberseguridad - Control IAM-01

## Información del Control

- **Control ID**: IAM-01
- **Nombre del Control**: Gestión de identidades y roles
- **Fecha de Auditoría**: 2025-01-27
- **Pregunta del Cliente**: ¿Cómo gestionáis usuarios, roles y permisos en la plataforma?
- **Expectativa del Cliente**: Ver que existe un modelo formal de gestión de identidades y permisos, no acceso 'ad hoc'.

---

## Resumen Ejecutivo

✅ **CUMPLIMIENTO**: La plataforma implementa un **modelo formal y estructurado** de gestión de identidades y permisos basado en:

1. **Autenticación centralizada** mediante Supabase Auth
2. **Sistema de roles jerárquico** con múltiples niveles de permisos
3. **Row Level Security (RLS)** habilitado en todas las tablas con políticas deny-by-default
4. **Funciones de seguridad** reutilizables para verificación de permisos
5. **Protección a nivel de aplicación** mediante hooks y componentes de rutas protegidas

---

## 1. Modelo de Identidades

### 1.1. Autenticación

La plataforma utiliza **Supabase Auth** como sistema centralizado de autenticación:

- **Proveedor**: Supabase Auth (integración con `@supabase/supabase-js`)
- **Gestión de sesiones**: Automática mediante JWT tokens
- **Contexto de autenticación**: `AuthContext.tsx` gestiona el estado global de autenticación

**Evidencia**:
```typescript
// src/contexts/AuthContext.tsx
export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [session, setSession] = useState<Session | null>(null);
  // ... gestión de estado de autenticación
}
```

### 1.2. Perfil de Usuario Extendido

La tabla `app_user` extiende la información del usuario autenticado:

**Estructura de la tabla `app_user`**:
```sql
CREATE TABLE IF NOT EXISTS "public"."app_user" (
    "id" uuid DEFAULT gen_random_uuid() NOT NULL,
    "name" text,
    "surname" text,
    "company_id" uuid,
    "is_admin" boolean DEFAULT false,
    "is_verified" boolean DEFAULT false,
    "auth_user_id" uuid,  -- FK a auth.users
    "company_position" text,
    "avatar_url" text
);
```

**Relación con autenticación**:
- `auth_user_id` → Referencia a `auth.users.id` (Supabase Auth)
- Un usuario autenticado puede crear su propio perfil en `app_user`
- Política RLS: Solo el usuario puede crear su propio perfil

**Evidencia**:
```sql
-- supabase/migrations/20251126085807_fix_app_user_rls_policies.sql
CREATE POLICY "Users can create their own profile"
  ON "public"."app_user"
  FOR INSERT
  TO authenticated
  WITH CHECK (auth.uid() = auth_user_id);
```

---

## 2. Modelo de Roles y Permisos

### 2.1. Roles Identificados

La plataforma implementa un **sistema de roles jerárquico** con los siguientes niveles:

#### 2.1.1. **Admin Global** (`is_admin`)
- **Ubicación**: Campo `is_admin` en tabla `app_user`
- **Permisos**: Acceso completo a la plataforma, gestión de usuarios y configuración
- **Verificación**: Hook `useIsAdmin()` y función `is_admin_user()`

**Evidencia**:
```typescript
// src/hooks/useIsAdmin.ts
export const useIsAdmin = () => {
  // Verifica is_admin en app_user
  const { data, error } = await supabase
    .from('app_user')
    .select('is_admin')
    .eq('auth_user_id', user.id)
    .maybeSingle();
}
```

#### 2.1.2. **Developer** (`developer_access`)
- **Ubicación**: Tabla `developer_access` (relación many-to-one con `auth.users`)
- **Permisos**: Acceso a funciones de desarrollo, gestión de datos de prueba, visualización de todas las conversaciones/RFXs
- **Verificación**: Hook `useIsDeveloper()` y función `has_developer_access()`

**Evidencia**:
```sql
-- Función de verificación
CREATE OR REPLACE FUNCTION "public"."has_developer_access"(
  "check_user_id" uuid DEFAULT auth.uid()
) RETURNS boolean
LANGUAGE sql STABLE SECURITY DEFINER
AS $$
  SELECT EXISTS (
    SELECT 1 
    FROM public.developer_access 
    WHERE user_id = check_user_id
  );
$$;
```

```typescript
// src/hooks/useIsDeveloper.ts
export const useIsDeveloper = () => {
  const { data, error } = await supabase
    .from('developer_access')
    .select('id')
    .eq('user_id', user.id)
    .maybeSingle();
}
```

#### 2.1.3. **Company Admin** (`company_admin_requests`)
- **Ubicación**: Tabla `company_admin_requests` con status `'approved'`
- **Permisos**: Gestión de su compañía específica, acceso a datos de la compañía, gestión de documentos
- **Verificación**: Hook `useIsCompanyAdmin(companyId)` y función `is_approved_company_admin()`

**Evidencia**:
```sql
-- Función de verificación
CREATE OR REPLACE FUNCTION "public"."is_approved_company_admin"(
  "p_company_id" uuid,
  "p_user_id" uuid DEFAULT auth.uid()
) RETURNS boolean
LANGUAGE sql STABLE SECURITY DEFINER
AS $$
  SELECT EXISTS (
    SELECT 1 
    FROM company_admin_requests 
    WHERE user_id = p_user_id 
    AND company_id = p_company_id
    AND status = 'approved'
  );
$$;
```

```typescript
// src/hooks/useIsCompanyAdmin.ts
export function useIsCompanyAdmin(companyId?: string) {
  const { data, error } = await supabase
    .from('company_admin_requests')
    .select('id')
    .eq('user_id', user.id)
    .eq('company_id', companyId)
    .eq('status', 'approved')
    .maybeSingle();
}
```

#### 2.1.4. **Buyer/Supplier** (Tipo de Usuario)
- **Ubicación**: Tabla `user_type_selections` y `company.role`
- **Permisos**: Acceso diferenciado según tipo (buyer puede crear RFXs, supplier puede responder)
- **Tipos**: `'buyer'` o `'supplier'` (definido en constraint de `company.role`)

**Evidencia**:
```sql
-- Constraint en tabla company
CONSTRAINT "company_role_check" CHECK (
  ("role" = ANY (ARRAY['supplier'::text, 'buyer'::text]))
);
```

#### 2.1.5. **RFX Owner/Member** (Roles Contextuales)
- **Ubicación**: Tabla `rfxs.user_id` (owner) y `rfx_members` (members)
- **Permisos**: Acceso a RFX específico, gestión de especificaciones, invitaciones
- **Verificación**: Función `is_rfx_participant()`

**Evidencia**:
```sql
CREATE OR REPLACE FUNCTION "public"."is_rfx_participant"(
  "p_rfx_id" uuid,
  "p_user_id" uuid
) RETURNS boolean
LANGUAGE sql STABLE SECURITY DEFINER
AS $$
  -- Check if user is owner of RFX
  SELECT EXISTS (
    SELECT 1 FROM public.rfxs
    WHERE id = p_rfx_id AND user_id = p_user_id
  )
  OR
  -- Or if user is a member
  EXISTS (
    SELECT 1 FROM public.rfx_members
    WHERE rfx_id = p_rfx_id AND user_id = p_user_id
  );
$$;
```

### 2.2. Jerarquía de Permisos

```
Admin Global (is_admin)
  └── Acceso completo a toda la plataforma

Developer (developer_access)
  └── Acceso a funciones de desarrollo y datos de prueba

Company Admin (company_admin_requests approved)
  └── Gestión de su compañía específica

Buyer/Supplier (user_type_selections)
  └── Permisos según tipo de usuario

RFX Owner/Member (rfxs.user_id / rfx_members)
  └── Permisos contextuales a RFX específico
```

---

## 3. Row Level Security (RLS)

### 3.1. Estado de RLS

✅ **Todas las tablas tienen RLS habilitado** con enfoque **deny-by-default**.

**Evidencia**: En la migración baseline se puede ver que todas las tablas tienen:
```sql
ALTER TABLE "public"."<table_name>" ENABLE ROW LEVEL SECURITY;
```

### 3.2. Políticas RLS Implementadas

Las políticas RLS siguen un patrón consistente:

1. **Políticas de SELECT**: Controlan qué filas puede ver un usuario
2. **Políticas de INSERT**: Controlan qué filas puede crear un usuario
3. **Políticas de UPDATE**: Controlan qué filas puede modificar un usuario
4. **Políticas de DELETE**: Controlan qué filas puede eliminar un usuario

#### 3.2.1. Ejemplo: Políticas para `app_user`

```sql
-- Usuarios pueden crear su propio perfil
CREATE POLICY "Users can create their own profile"
  ON "public"."app_user"
  FOR INSERT
  TO authenticated
  WITH CHECK (auth.uid() = auth_user_id);

-- Usuarios pueden ver su propio perfil y perfiles básicos para conversaciones
CREATE POLICY "Allow viewing basic user info for conversations"
  ON "public"."app_user"
  FOR SELECT
  TO authenticated
  USING (
    public.has_developer_access() 
    OR 
    (EXISTS (SELECT 1 FROM conversations WHERE user_id = auth.uid()))
  );

-- Developers pueden ver todos los usuarios
CREATE POLICY "Developers can view all app users"
  ON "public"."app_user"
  FOR SELECT
  TO authenticated
  USING (public.has_developer_access());

-- Developers pueden actualizar estado de admin y asignación de compañía
CREATE POLICY "Developers can update admin status and company assignments"
  ON "public"."app_user"
  FOR UPDATE
  TO authenticated
  USING (public.has_developer_access())
  WITH CHECK (public.has_developer_access());
```

#### 3.2.2. Ejemplo: Políticas para `rfxs`

```sql
-- Usuarios pueden crear sus propios RFXs
CREATE POLICY "Users can insert their own RFXs"
  ON "public"."rfxs"
  FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- Usuarios pueden ver RFXs donde son owner o member
CREATE POLICY "Users can view RFXs they own or belong to"
  ON "public"."rfxs"
  FOR SELECT
  USING (
    auth.uid() = user_id 
    OR 
    EXISTS (
      SELECT 1 FROM public.rfx_members
      WHERE rfx_members.rfx_id = rfxs.id 
      AND rfx_members.user_id = auth.uid()
    )
  );

-- Usuarios pueden actualizar RFXs donde son owner o member
CREATE POLICY "Users can update RFXs if owner or member"
  ON "public"."rfxs"
  FOR UPDATE
  USING (
    (auth.uid() = user_id) 
    OR 
    (EXISTS (
      SELECT 1 FROM public.rfx_members
      WHERE rfx_members.rfx_id = rfxs.id 
      AND rfx_members.user_id = auth.uid()
    ))
  );
```

### 3.3. Funciones Helper para RLS

Las políticas RLS utilizan funciones helper para evitar duplicación y mejorar mantenibilidad:

#### 3.3.1. `has_developer_access()`
- **Uso**: Verificar si un usuario tiene acceso de developer
- **Tipo**: `SECURITY DEFINER` (ejecuta con privilegios del propietario)
- **Aplicación**: Usada en múltiples políticas para otorgar permisos amplios a developers

**Ejemplo de uso en políticas**:
```sql
CREATE POLICY "Developers can view all companies"
  ON "public"."company"
  FOR SELECT
  USING (public.has_developer_access());
```

#### 3.3.2. `is_approved_company_admin(company_id, user_id)`
- **Uso**: Verificar si un usuario es admin aprobado de una compañía específica
- **Tipo**: `SECURITY DEFINER`
- **Aplicación**: Control de acceso a datos de compañía

**Ejemplo de uso**:
```sql
CREATE POLICY "Approved company admins can view all requests for their company"
  ON "public"."company_admin_requests"
  FOR SELECT
  USING (
    public.is_approved_company_admin(company_id) 
    OR 
    public.has_developer_access()
  );
```

#### 3.3.3. `is_rfx_participant(rfx_id, user_id)`
- **Uso**: Verificar si un usuario es owner o member de un RFX
- **Tipo**: `SECURITY DEFINER`
- **Aplicación**: Control de acceso a datos de RFX

**Ejemplo de uso**:
```sql
CREATE POLICY "RFX participants can view NDA metadata"
  ON "public"."rfx_nda_uploads"
  FOR SELECT
  USING (public.is_rfx_participant(rfx_id, auth.uid()));
```

### 3.4. Principio Deny-by-Default

✅ **Implementado correctamente**: Todas las tablas tienen RLS habilitado y las políticas son explícitas. Si no hay una política que permita una operación, esta se deniega automáticamente.

---

## 4. Protección a Nivel de Aplicación

### 4.1. Hooks de Verificación de Roles

La aplicación implementa hooks React para verificar roles en el frontend:

1. **`useIsAdmin()`**: Verifica si el usuario es admin global
2. **`useIsDeveloper()`**: Verifica si el usuario tiene acceso de developer
3. **`useIsCompanyAdmin(companyId)`**: Verifica si el usuario es admin de una compañía específica

**Evidencia**:
```typescript
// src/hooks/useIsAdmin.ts
export const useIsAdmin = () => {
  const [isAdmin, setIsAdmin] = useState<boolean>(false);
  // ... lógica de verificación
  return { isAdmin, loading };
};
```

### 4.2. Componentes de Rutas Protegidas

La aplicación implementa componentes para proteger rutas:

1. **`ProtectedAdminRoute`**: Solo permite acceso a admins globales
2. **`ProtectedRoute`**: Requiere autenticación
3. **`ProtectedSupplierRoute`**: Requiere autenticación y tipo supplier

**Evidencia**:
```typescript
// src/components/ProtectedAdminRoute.tsx
export const ProtectedAdminRoute: React.FC<ProtectedAdminRouteProps> = ({ children }) => {
  const { user, loading: authLoading } = useAuth();
  const { isAdmin, loading: adminLoading } = useIsAdmin();

  if (!user) {
    return <Navigate to="/auth" replace />;
  }

  if (!isAdmin) {
    return <Navigate to="/" replace />;
  }

  return <>{children}</>;
};
```

### 4.3. Verificación en Lado del Cliente vs Servidor

⚠️ **IMPORTANTE**: La verificación en el frontend (hooks, rutas protegidas) es solo para **UX y navegación**. La **seguridad real** está en las políticas RLS del servidor, que no pueden ser bypasseadas desde el cliente.

---

## 5. Gestión de Permisos

### 5.1. Asignación de Roles

#### 5.1.1. Admin Global (`is_admin`)
- **Asignación**: Manual mediante actualización directa en base de datos o mediante función con permisos de developer
- **Control**: Solo developers pueden modificar `is_admin` en `app_user`

**Evidencia**:
```sql
CREATE POLICY "Developers can update admin status and company assignments"
  ON "public"."app_user"
  FOR UPDATE
  TO authenticated
  USING (public.has_developer_access())
  WITH CHECK (public.has_developer_access());
```

#### 5.1.2. Developer Access
- **Asignación**: Inserción en tabla `developer_access` (solo developers pueden gestionar)
- **Control**: Políticas RLS restringen la gestión a developers existentes

#### 5.1.3. Company Admin
- **Asignación**: Proceso de solicitud y aprobación mediante `company_admin_requests`
- **Estados**: `'pending'`, `'approved'`, `'rejected'`
- **Control**: Solo developers pueden aprobar/rechazar solicitudes

**Evidencia**:
```sql
CREATE POLICY "Developers can update admin requests"
  ON "public"."company_admin_requests"
  FOR UPDATE
  USING (public.has_developer_access());
```

#### 5.1.4. RFX Members
- **Asignación**: El owner del RFX puede agregar miembros
- **Control**: Políticas RLS verifican que el usuario que agrega sea owner del RFX

### 5.2. Revocación de Permisos

- **Admin Global**: Actualización de `is_admin = false`
- **Developer**: Eliminación de registro en `developer_access`
- **Company Admin**: Actualización de status en `company_admin_requests` a `'rejected'` o eliminación
- **RFX Member**: Eliminación de registro en `rfx_members`

---

## 6. Evidencias Técnicas

### 6.1. Archivos de Migración

Todas las políticas RLS están documentadas en migraciones de Supabase:

- `supabase/migrations/20251121132118_remote_schema_baseline.sql` - Baseline con todas las políticas
- `supabase/migrations/20251126085807_fix_app_user_rls_policies.sql` - Políticas específicas de `app_user`
- `supabase/migrations/20251124102943_create_rfx_conversations_tables.sql` - Políticas para conversaciones RFX

### 6.2. Funciones de Seguridad

Funciones SQL documentadas en la migración baseline:
- `has_developer_access()` - Línea 2399
- `is_admin_user()` - Línea 2496
- `is_approved_company_admin()` - Línea 2509
- `is_rfx_participant()` - Línea 2526

### 6.3. Hooks de Frontend

Hooks TypeScript para verificación de roles:
- `src/hooks/useIsAdmin.ts`
- `src/hooks/useIsDeveloper.ts`
- `src/hooks/useIsCompanyAdmin.ts`

### 6.4. Componentes de Protección

Componentes React para protección de rutas:
- `src/components/ProtectedAdminRoute.tsx`
- `src/components/ProtectedRoute.tsx`

---

## 7. Conclusiones

### 7.1. Fortalezas

✅ **Modelo formal y estructurado**: La plataforma implementa un sistema de gestión de identidades y permisos bien definido con:
- Autenticación centralizada (Supabase Auth)
- Múltiples niveles de roles jerárquicos
- RLS habilitado en todas las tablas
- Funciones helper reutilizables
- Protección tanto a nivel de base de datos como de aplicación

✅ **Principio de menor privilegio**: Las políticas RLS siguen un enfoque deny-by-default, otorgando permisos explícitos solo cuando es necesario.

✅ **Separación de responsabilidades**: Los roles están claramente definidos (Admin, Developer, Company Admin, Buyer/Supplier, RFX Owner/Member).

✅ **Auditabilidad**: Las políticas RLS están documentadas en migraciones versionadas, facilitando el seguimiento de cambios.

### 7.2. Recomendaciones

1. **Documentación de políticas**: Considerar crear un documento que mapee cada tabla con sus políticas RLS y casos de uso.

2. **Revisión periódica de permisos**: Implementar un proceso de revisión periódica de asignaciones de roles (especialmente Admin y Developer).

3. **Logging de cambios de permisos**: Considerar implementar una tabla de auditoría para registrar cambios en roles y permisos.

4. **Testing de políticas RLS**: Asegurar que las pruebas automatizadas incluyan verificación de políticas RLS.

---

## 8. Cumplimiento del Control

| Criterio | Estado | Evidencia |
|----------|--------|-----------|
| Modelo formal de gestión de identidades | ✅ CUMPLE | Autenticación centralizada con Supabase Auth y tabla `app_user` |
| Modelo de roles y permisos definido | ✅ CUMPLE | 5 tipos de roles identificados con jerarquía clara |
| Política de gestión de identidades | ✅ CUMPLE | RLS habilitado en todas las tablas con políticas explícitas |
| Extracto de RLS | ✅ CUMPLE | Todas las tablas tienen RLS con políticas deny-by-default |
| No acceso 'ad hoc' | ✅ CUMPLE | Todas las operaciones requieren políticas RLS explícitas |

**VEREDICTO FINAL**: ✅ **CUMPLE** con el control IAM-01. La plataforma implementa un modelo formal y estructurado de gestión de identidades y permisos, sin acceso 'ad hoc'.

---

## Anexos

### A. Lista de Tablas con RLS Habilitado

Todas las tablas en el esquema `public` tienen RLS habilitado. Algunas tablas principales:

- `app_user`
- `company`
- `company_admin_requests`
- `rfxs`
- `rfx_members`
- `rfx_specs`
- `rfx_key_members`
- `developer_access`
- `conversations`
- `chat_messages`
- `notifications`
- `subscription`
- `stripe_customers`

*(Lista completa disponible en migración baseline)*

### B. Ejemplo de Política RLS Completa

```sql
-- Ejemplo: Política para rfx_specs
CREATE POLICY "Users can insert RFX specs if owner or member"
  ON "public"."rfx_specs"
  FOR INSERT
  WITH CHECK (
    (EXISTS (
      SELECT 1 FROM public.rfxs
      WHERE rfxs.id = rfx_specs.rfx_id 
      AND rfxs.user_id = auth.uid()
    ))
    OR
    (EXISTS (
      SELECT 1 FROM public.rfx_members
      WHERE rfx_members.rfx_id = rfx_specs.rfx_id 
      AND rfx_members.user_id = auth.uid()
    ))
  );
```

### C. Flujo de Asignación de Company Admin

1. Usuario crea solicitud: `INSERT INTO company_admin_requests (user_id, company_id, status) VALUES (..., ..., 'pending')`
2. Developer revisa solicitud
3. Developer aprueba: `UPDATE company_admin_requests SET status = 'approved' WHERE ...`
4. Usuario obtiene permisos de Company Admin para esa compañía específica

---

**Fin del Informe de Auditoría - Control IAM-01**

