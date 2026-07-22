---
id: DP.FM.294
name: "Android safe-area: захардкоженная высота нижнего бара наезжает на системную навигацию"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-19
source: "session-close 2026-07-19; WP-32 (DS-humdes/mobile, коммиты f623818 + eecfec3)"
related:
  see_also:
    - "DP.FM.293: RNGH глобальная замена (тот же WP-33, смежный Android-класс)"
tags: [react-native, android, safe-area, bottom-inset, navigation-bar, useSafeAreaInsets, hardcoded]
---

# DP.FM.294 — Android safe-area: захардкоженная высота нижнего бара наезжает на системную навигацию

## Паттерн

Компоненты нижней границы экрана (таб-бар, bottom sheet) используют захардкоженные значения высоты (например, `height: 12`) вместо динамического `useSafeAreaInsets().bottom` — контент наезжает на системную навигацию Android (кнопки «Назад/Дом/Недавние» или home indicator).

## Диагностика

**Тест:** «Нижний контент перекрывается системными кнопками Android на устройствах с жестовой или кнопочной навигацией?» Да → пропущен `useSafeAreaInsets().bottom`.

**Инструмент:** проверять на release APK (не debug — debug требует Metro, на Android падает «unable to load script»). Debug-сборка ненадёжна для проверки safe-area.

**Два независимых компонента, один класс ошибки:**
1. `BottomTabBar` — spacer с `height: 12` вместо `height: bottom`
2. `UniversalBottomSheet` — `childrenStyle` без `paddingBottom: bottom` вообще

## Инцидент

WP-32 (DS-humdes/mobile, 19 июля 2026):
- `BottomTabBar.component.tsx` — `f623818`: спейсер `height: bottom` из `useSafeAreaInsets()`
- `UniversalBottomSheet.component.tsx` — `eecfec3`: `childrenStyle` с `paddingBottom: bottom`

Оба подтверждены пилотом на реальном устройстве через release APK.

## Fix

```tsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function BottomTabBar() {
  const { bottom } = useSafeAreaInsets();
  return (
    <View>
      {/* ... контент ... */}
      <View style={{ height: bottom }} /> {/* вместо height: 12 */}
    </View>
  );
}

function UniversalBottomSheet({ children, childrenStyle, ...props }) {
  const { bottom } = useSafeAreaInsets();
  return (
    <Modalize
      childrenStyle={[{ paddingBottom: bottom }, childrenStyle]}
      {...props}
    >
      {children}
    </Modalize>
  );
}
```

## Правило

**Все компоненты, касающиеся нижней границы экрана, обязаны получать `bottom = useSafeAreaInsets().bottom` и применять его как `paddingBottom`/`height`.**

Захардкоженная высота нижнего отступа = гарантированная регрессия на Android с жестовой навигацией.

## Применимость

Любой React Native проект под Android. Особо уязвимы:
- Таб-бары (фиксированный нижний UI)
- Bottom sheets (Modalize, `@gorhom/bottom-sheet`)
- FAB-кнопки и любые элементы `position: absolute, bottom: 0`