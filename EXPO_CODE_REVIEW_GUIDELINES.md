# Code Rules

- **Pressable > TouchableOpacity**: Siempre usar `Pressable` para elementos interactivos
- **StyleSheet > estilos inline**: usar `StyleSheet.create()` siempre. Solo usar estilos inline para valores dinámicos que cambian en el componente
- Navegación con `expo-router`: preferir `useRouter()` hook. Ej: `const { navigate, push } = useRouter()`
- Iconos: `@expo/vector-icons` (Ionicons)
- **Imports con alias**: siempre usar `@/src/...` (ej: `@/src/components`, `@/src/screens`, `@/src/theme`)
- **No usar Barrel Exports**: importar cada componente desde su carpeta, no desde un index común
  - ✅ `import { Button } from '@/src/components/Button'`
  - ❌ `import { Button, Text } from '@/src/components'`
- **FlashList > FlatList**: siempre usar `FlashList` de `@shopify/flash-list` para renderizar listas (más performante)
- **Reanimated > Animated**: usar `react-native-reanimated` para animaciones. Solo usar `Animated` de RN si no es posible con Reanimated
- **Animaciones ≤ 300ms**: duración máxima por defecto, salvo que se indique lo contrario
- **Image de Expo**: usar `Image` de `expo-image`, no de `react-native`
- **MMKV > AsyncStorage**: usar `react-native-mmkv` para almacenamiento local, nunca AsyncStorage

## Estructura

### Rutas (file-based routing)

- `app/` contiene archivos de ruta (expo-router) y layouts de configuración
- Los archivos de pantalla en `app/` solo exportan desde `src/screens/` (sin lógica propia)
- Los `_layout.tsx` sí tienen contenido propio (configuración de navegación)
- Ejemplo: `app/auth/sign-in.tsx` → `export { default } from '@/src/screens/auth/sign-in'`

### Pantallas

- Contenido real en: `src/screens/[ruta]/index.tsx`
- Ejemplo: `app/auth/sign-in.tsx` → `src/screens/auth/sign-in/index.tsx`
- Carpetas hermanas opcionales dentro de la pantalla:
  - `interfaces/` - tipos e interfaces de la pantalla
  - `components/` - componentes exclusivos de esa pantalla
  - `constants/` - constantes locales

### Componentes compartidos

- `src/components/NombreComponente/index.tsx`

### Convención de nombres

- Archivos y carpetas en `app/` → **kebab-case** (ej: `sign-in.tsx`, `forgot-password.tsx`)
- Carpetas en `src/screens/` → **kebab-case** (coinciden con la ruta en app). Ej: `src/screens/auth/sign-in/`
- Componentes (archivos `.tsx` dentro de carpetas) → **PascalCase** (ej: `LoginForm.tsx`)

### Estructura interna del archivo

- Orden: imports → componente → styles → export default
- El `export default` siempre va **debajo** de los estilos (corregir si está arriba)

## TypeScript

- **Nunca usar `any`**: siempre usar tipados adecuados
- Tipar parámetros de funciones y valores de retorno
- Interfaces para props de componentes
- Tipar estados con useState

## Seguridad

- Nunca loguear datos sensibles (passwords, tokens)
- Limpiar passwords del estado después de uso

## Accesibilidad

- `accessibilityLabel` en elementos interactivos
- `accessibilityRole` apropiado

## Comentarios

- **Siempre en inglés**: todos los comentarios en el código deben estar escritos en inglés

## Diseño

- Si hay un archivo Figma enlazado al proyecto, usarlo como referencia
- Los valores de Figma pueden diferir de los tokens del theme (ajustar según diseño)
