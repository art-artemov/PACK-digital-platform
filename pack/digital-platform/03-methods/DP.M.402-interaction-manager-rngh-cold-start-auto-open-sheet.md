---
id: DP.M.402
name: "InteractionManager.runAfterInteractions для RNGH-тачаблов в авто-открываемой шторке"
type: method
domain: digital-platform / mobile
status: active
valid_from: 2026-07-19
sources:
  - "WP-33 (DS-humdes/mobile), session-close 2026-07-19; коммиты a3e9823 + 180ad17"
related:
  see_also:
    - "DP.FM.293: Глобальная замена RNGH — gesture conflict (другая природа того же класса бага)"
tags: [react-native, android, rngh, interaction-manager, modalize, cold-start, timing]
---

# DP.M.402 — InteractionManager.runAfterInteractions для RNGH-тачаблов в авто-открываемой шторке

## Определение

Метод устранения «холодного старта» RNGH-тачаблов в шторке, которая открывается программно (не по тапу пользователя): откладывает рендер интерактивных элементов до завершения инициализации gesture handler'ов через `InteractionManager.runAfterInteractions`.

## Контекст применения

**Проблема:** шторка (`Modalize`) открывается автоматически при монтировании экрана. RNGH ещё не завершил setup gesture handler'ов. Тачаблы внутри не реагируют на первый тап — «холодный старт».

**Симптом:** первый тап после программного открытия шторки игнорируется, второй работает.

Отличается от DP.FM.293 (арбитраж жестов): там причина — конфликт RNGH-контекстов. Здесь — timing инициализации.

## IPO

- **Вход:** RNGH-тачаблы внутри шторки, открывающейся программно при монтировании
- **Процесс:** откладывать рендер интерактивного контента до `runAfterInteractions`
- **Выход:** тачаблы доступны после завершения setup RNGH, первый тап работает

## Реализация

```tsx
function AutoOpenSheetContent() {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    const interaction = InteractionManager.runAfterInteractions(() => {
      setReady(true);
    });
    return () => interaction.cancel();
  }, []);

  if (!ready) return null; // или skeleton/placeholder

  return (
    <GestureHandlerTouchableOpacity onPress={...}>
      {/* контент */}
    </GestureHandlerTouchableOpacity>
  );
}
```

## Когда применять

- Шторка открывается **программно** (не по тапу) при монтировании экрана или по условию
- Внутри шторки есть RNGH-тачаблы (opt-in через флаг, см. [DP.FM.293](../05-failure-modes/DP.FM.293-rngh-global-replace-gesture-conflict.md))
- Симптом: первый тап не работает, последующие — работают

Если первый тап работает — этот метод не нужен.

## Источник

WP-33 (DS-humdes/mobile, 19 июля 2026): 7-е место тач-арбитраж-бага имело другую природу — программно открываемая шторка; `InteractionManager.runAfterInteractions` подтверждён пилотом на реальном устройстве.