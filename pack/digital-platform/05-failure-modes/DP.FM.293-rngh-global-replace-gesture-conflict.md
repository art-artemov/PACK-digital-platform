---
id: DP.FM.293
name: "Глобальная замена тачаблов на RNGH ломает жестовую обработку в несвязанных компонентах"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-19
source: "session-close 2026-07-19; WP-33 (DS-humdes/mobile)"
related:
  see_also:
    - "DP.FM.294: Android safe-area — захардкоженная высота (тот же WP-33)"
    - "DP.M.402: InteractionManager cold-start для авто-открываемых шторок (дополняет)"
tags: [react-native, android, rngh, gesture-handler, modalize, touchable, regression, opt-in]
---

# DP.FM.293 — Глобальная замена тачаблов на RNGH ломает жестовую обработку в несвязанных компонентах

## Паттерн

Попытка заменить все `Button`/`ColClickable` в React Native проекте на `RNGH TouchableOpacity/Pressable` единым коммитом вызывает регрессию: RNGH-тачаблы внутри `Modalize`-шторок начинают конфликтовать с жестами самой шторки на Android — тачаблы перестают реагировать на тапы.

Симптом маскируется: в iOS и вне шторок всё работает, регрессия видна только на Android внутри модальных компонентов.

## Диагностика

**Тест:** «Тачаблы внутри `Modalize` не реагируют на тапы после RNGH-замены, хотя снаружи работают?» Да → конфликт жестового арбитража RNGH с gesture handler'ами Modalize.

**Механизм:** RNGH-тачаблы корректно работают только внутри одного `GestureHandlerRootView`-контекста. `Modalize` устанавливает собственный контекст жестов — при добавлении RNGH-тачаблов снаружи этого контекста (глобальная замена) арбитраж разрушается.

## Инцидент

WP-33 (DS-humdes/mobile, 19 июля 2026): коммит `10f1c26` заменил все `Button`/`ColClickable` на RNGH Touchable. После проверки на устройстве пилот обнаружил регрессию на Android — элементы внутри шторок перестали реагировать. Откат `c55748a`, переход на opt-in паттерн `c8ae158`.

## Fix

Точечная замена через **opt-in проп** на каждом компоненте:

```tsx
// В Button/ColClickable добавляется флаг:
interface ButtonProps {
  useGestureTouchable?: boolean;
  // ...
}

function Button({ useGestureTouchable, ...props }: ButtonProps) {
  const Touchable = useGestureTouchable
    ? GestureHandlerTouchableOpacity  // RNGH
    : TouchableOpacity;               // базовый RN
  return <Touchable {...props} />;
}

// Использование (только в конкретном месте, где нужен RNGH):
<Button useGestureTouchable onPress={...} />
```

7 мест исправлено поштучно — только тачаблы, где требовался тач-арбитраж Modalize/Android.

## Правило

**В React Native проекте с Modalize/RNGH переходить на RNGH-тачаблы только per-component opt-in, не глобальной заменой.**

Глобальная замена через всю кодовую базу = регрессия жестов на Android внутри модальных компонентов.

## Применимость

React Native проекты с `react-native-modalize` (или аналогами: `@gorhom/bottom-sheet`, `react-native-reanimated`), использующими собственный жестовый контекст.