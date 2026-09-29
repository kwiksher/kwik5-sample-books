# Interaction Unit Test Spec

How the editor UI dispatches pointer events to a layer object, and how to write
`Solar2D/Test/interaction/suite_*.lua` tests that a coding agent can generate
mechanically.

Source of truth read while writing this: `Test/helper.lua`, `Test/base_suite.lua`,
`Test/index.lua`, `lua_modules/kwiksher/kwik/editor/parts/layerTable.lua`,
`.../layerTableCommands.lua`, `.../baseTable.lua`, `.../selectorBase.lua`,
`.../buttons.lua`, `.../lua_modules/kwiksher/kwik/components/kwik/layer_*.lua`.

---

## 1. The three dispatch channels

Every interaction in the editor is one of three calls on a display object. There
is no OS-level mouse; the tests synthesize the Solar2D event directly.

| # | Intent | Exact call | Runtime handler that fires | Event fields |
|---|--------|-----------|----------------------------|--------------|
| 1 | Left click / touch select | `obj:touch{phase="ended"}` | `obj.touch`, registered with `obj:addEventListener("touch", obj)` | `phase="ended"` only |
| 2 | Right click (context menu) | `obj:dispatchEvent{name="mouse", target=obj, isSecondaryButtonDown=true, x=obj.x, y=obj.y}` | `mouseHandler`, registered with `obj:addEventListener("mouse", handler)` | `name="mouse"`, `target`, `isSecondaryButtonDown`, `x`, `y` |
| 3 | Selector row activation | `obj:tap{target=obj, eventName=...}` or `obj:dispatchEvent{name="tap", target=obj}` | `obj.tap`, registered with `obj:addEventListener("tap", obj)` | `target`, plus any extra keys the handler reads |

`touch` and `tap` are two different event names. Rows created by
`selectorBase.lua` listen for `"tap"`; rows created by `layerTable.lua` and
`baseTable.lua` listen for `"touch"`. Dispatching the wrong name silently does
nothing.

`obj:dispatchEvent{name="touch", phase="ended", target=obj}` also works (both
paths reach `obj.touch`), but every suite in the repo uses the `obj:touch{...}`
shortcut. Prefer the shortcut.

### 1.1 What "the layer object" is

`helper.selectLayer(name)` returns the row object, not the scene object.

- `obj.text` - the label shown in the table (may carry `"├ "` / `"└ "` prefixes
  for nested layers; `obj.layer` has the clean name).
- `obj.layer` - the layer name used by `UI.sceneGroup[layer]`.
- `obj.class`, `obj.name`, `obj.suffix` - class metadata for the row.
- `obj.rect` - the hit rect behind the row; selection color is set on this.
- `obj.parentObj` - parent row when nested.
- `obj.classEntries` - per-class child rows (icons or text) inside the layer row.
- `obj.isSelected`, `obj.shapedWith`, `obj.isIndex`, `obj.childEntries`.

The scene object (the actual sprite on the page) is reached separately:

```lua
local sceneObj = M.UI.sceneGroup[layerName]          -- flat layer
local sceneObj = M.UI.sceneGroup["parent/child"]     -- nested path via util.getLayerPath
```

### 1.2 Nested layer addressing

`helper.selectLayer` splits the lookup name on `" / "`:

```lua
helper.selectLayer("groupA/childB")          -- resolves against obj.parentObj.layer
helper.selectLayer("sprite", "sprite")       -- 2nd arg selects a classEntries child
helper.selectLayer("sprite", "linear")       -- class match inside the layer row
helper.selectLayer("sprite", nil, true)      -- right-click the layer row
helper.selectLayer("sprite", "linear", true) -- right-click the class child
```

When `class` is given, the dispatch targets `obj.classEntries[i]` (the class
child), otherwise it targets the layer row.

---

## 2. Exact helper API (Test/helper.lua)

All helpers require `helper.init(props)` to have run, which `base_suite.lua`
does inside `M.init`.

### Selection and click

```lua
helper.selectLayer(name)                       -- touch ended on layer row; returns obj
helper.selectLayer(name, class)                -- touch ended on class child; returns classObj
helper.selectLayer(name, class, isRightClick)  -- mouse dispatch instead of touch
helper.selectGroup(name[, class[, isRightClick]])
helper.selectAsset(name[, isRightClick])       -- tap if obj.touch missing, else touch
helper.selectAudio(name[, isRightClick])
helper.selectAction(name[, isRightClick])
helper.selectVariable(name[, isRightClick])
helper.selectBook(name[, isRightClick])        -- mouse only (no touch fallback)
helper.selectPage(name[, isRightClick])
helper.selectComponent(name[, isRightClick])   -- componentSelector rows
helper.selectAssetIcon(name)                   -- dispatchEvent{name="tap"}
helper.selectComponentIcon(name)               -- dispatchEvent{name="tap"}
helper.selectActionGroup(name)                 -- actionController.commandGroupHandler
helper.selectActionCommand(class, name)        -- commandbox row :tap{numTaps=1}
helper.selectEntries(box, names)               -- box.objs[v].layer == n -> v:touch{phase="ended"}
helper.selectTimer(name[, isRightClick])
```

### Buttons, props, assets

```lua
helper.clickButton(name[, buttonsContext])     -- matches v.eventName; prefers rect:touch
helper.clickButtonInRow(parent, name)          -- nested row buttons, uses rect:tap()
helper.clickProp(objs, name)                   -- matches v.text; dispatchEvent{name="tap", ...}
helper.clickObj = helper.clickProp
helper.setProp(objs, name, value)              -- writes obj.field.text = value
helper.clickAsset(objs, name)                  -- obj:touch{phase="ended"}
helper.clickAction(name)                       -- actionTable.objs[v].text == name -> v:touch
helper.clickIcon(toolGroup, tool)              -- toolbar tool, deferred by 1000ms
helper.clickIconObj(tbl, name)                 -- iconObjs callBack
helper.selectIcon(toolGroup, tool)             -- returns a promise (see 2.1)
```

### 2.1 Async helpers return Deferred promises

`helper.selectIcon` resolves on a `timer.performWithDelay(500, ...)`. Chain it:

```lua
helper.selectIcon("Interactions", "Button"):done(function() end)
```

`helper.clickIcon` schedules its inner click 1000ms out and returns nothing.
Suites that need ordering use `timer.performWithDelay` directly (see
`Test/page/suite_variable.lua`).

### 2.2 Introspection

```lua
helper.hasObj(layerTable, "name"[, class])   -- booleans
helper.getObj(objs, name)
helper.getObjs(t, names)
helper.getEntries(box, names)
helper.getPage(name[, isRightClick])
helper.getFillColor(object)                  -- json decodes object._properties
helper.extractProps(props)                   -- selectors, UI, bookTable, pageTable, layerTable
helper.init(props)
helper.initSuite(props, extras)
```

---

## 3. Suite contract

A suite is a plain table of functions named `test*`. `lunatest` collects only
keys where `k:sub(1,4) == "test"` (see `lua_modules/lunatest.lua:566`). Rename a
test to `xtest_*` to keep it in the file but out of the run, which is the
convention used throughout this repo.

```lua
local M = require("Test.base_suite").new({
  selectApp = true,           -- dispatch editor.selector.selectApp in suite_setup
  book = "bookFree",          -- bookTable.commandHandler
  page = "page1",             -- pageTable.commandHandler
  component = "withClick",    -- "iconOnly" skips componentSelector:onClick
  componentTable = "layerTable",
})

local helper = require("Test.helper")

function M.setup() end        -- before each test
function M.teardown() end     -- after each test
function M.suite_setup() end  -- once before the suite; base_suite provides a default

function M.test_something()
  ...
end

return M
```

`base_suite.new()` preloads `M.json, M.groupTable, M.buttons, M.actionTable,
M.commandbox, M.actionCommandPropsTable, M.actionController, M.assetTable,
M.variableTable, M.audioTable, M.actionCommandTable, M.actionEditor,
M.partsButtons, M.classProps, M.actionbox, M.actionButtons,
M.actionButtonContext, M.actionCommandButtons, M.actionboxButtonContext,
M.listbox, M.listPropsTable, M.listButtons, M.picker, M.scriptsCommands,
M.util, M.editorUtil, M.colorPicker, M.timerTable` and sets
`M.selectors, M.UI, M.bookTable, M.pageTable, M.layerTable` in `M.init`.

Register the suite in `Test/index.lua` under the `set(book, page)` matching the
running `env.book` / `env.page`:

```lua
set("interaction", "swipe")   -- runs Test.interaction.suite_swipe
```

`set` also requires that the running page matches: `pagePart = page:match("([^_]+)")`
compared to `UI.page`.

Assertions come from `lunatest` as globals (no require):
`assert_true, assert_false, assert_nil, assert_not_nil, assert_equal,
assert_not_equal, assert_gt, assert_gte, assert_lt, assert_lte, assert_len,
assert_not_len, assert_match, assert_not_match, assert_boolean, assert_number,
assert_string, assert_table, assert_function, assert_userdata, assert_metatable,
assert_error, assert_random`. Note `assert_equal(exp, got)` takes expected first
(same for `assert_gt(lim, val)` and friends).

Skip a case with `skip("reason")`.

---

## 4. Reference sequences

### 4.1 Select a layer, then open its class editor (alt-down)

```lua
function M.test_edit_filter()
  local name = "rect_0"
  M.layerTable.altDown = true
  local obj = helper.selectLayer(name, "filter")
  M.layerTable.altDown = false

  assert_not_nil(obj)
  assert_equal(obj.class, "filter")
end
```

`Test/animation/suite_filter.lua` is the working example. `altDown` is read by
`layerTable:isAltDown()` -> `layerTableCommands.commandHandler`, which routes to
`showLayerProps` / `showClassProps` instead of plain selection.

### 4.2 Right-click context menu

```lua
local obj = helper.selectLayer(name, nil, true)
assert_true(obj.isSelected == nil or obj.isSelected == true)
assert_not_nil(M.buttons.contextMenuOptions)
```

`layerTableCommands.mouseHandler` only shows the menu when
`event.isSecondaryButtonDown and event.target.isSelected`. So select first
(plain touch), then right-click, when the target is not already selected.
`buttons:showContextMenu` stores `self.contextMenuOptions` and
`self.contextButtons`; `buttons:hideContextMenu()` clears them.

### 4.3 Multi-select with control down

```lua
M.layerTable.controlDown = true
helper.selectLayer("title")
helper.selectLayer("gotoBtn")
M.layerTable.controlDown = false
assert_len(2, M.layerTable.selections)
```

### 4.4 Drive a scene-layer interaction (runtime, not editor)

Editor tables drive the editor; the page layers are driven through
`M.UI.sceneGroup`. Runtime handlers are registered in
`lua_modules/kwiksher/kwik/components/kwik/layer_*.lua`.

| Interaction | Listener | Synthetic event | Emulator helper |
|---|---|---|---|
| Button | `addEventListener("tap", obj)` | `sceneObj:tap{numTaps=1, target=sceneObj}` | none |
| Canvas | `addEventListener("touch", fn)` | `sceneObj:dispatchEvent{name="touch", phase="began"/"moved"/"ended", target=sceneObj, x=, y=}` | none |
| Drag / Pinch / Spin | `dmc_multitouch` registers `addEventListener("touch", cb)` | `sceneObj:dispatchEvent{name="touch", phase=, id=, target=sceneObj, x=, y=}` | `layer_pinch.createPinchEmulator(target, cb)` |
| Shake | `addEventListener("accelerometer", fn)` | `obj.accelerometer` payload | `layer_shake.createShakeEmulator(obj, cb)` |
| Swipe | `Gesture.SWIPE_EVENT` | `{phase="ended", direction="left"/"right"/"up"/"down"}` | none yet |
| Parallax | accelerometer-ish tick | dispatcher tick | `layer_parallax.dummyDispatcher(targets, interval)` |

Two rules for scene objects, both verified on a live simulator:

- `sceneObj:touch{...}` is **not** an event channel. It is a method call on a
  field named `touch`; scene objects normally have no such field, so it raises
  `attempt to call method 'touch' (a nil value)`. Use `dispatchEvent` instead.
  The `obj:touch{phase=...}` form in this document is valid only for editor
  table rows, which do define a `touch` function.
- A tap still needs `numTaps` (it is filtered against the layer's
  `properties.btaps`) and `target` (action commands read `event.target`).

For `dmc_multitouch` interactions the listener is registered under the plain
`"touch"` name via `TouchMgr:register`, which does
`obj:addEventListener("touch", callback)`
(`lua_modules/dmc_touchmanager.lua:202-213`). So a synthesized touch reaches
drag/pinch/spin too. `event.id` is tracked in `dmc.touches` and
`dmc.touchStack`, so supply a stable `id`.

Shake example, verbatim shape from `Test/interaction/suite_shake.lua`:

```lua
function M.test_emulator()
  local mod = require("components.kwik.layer_shake")
  local obj = M.UI.sceneGroup["ellipse_0"]
  if obj == nil then
    assert_not_nil(obj, "scene object ellipse_0 not found; check env.book/env.page")
    return
  end
  mod.createShakeEmulator(obj, function(event)
    event.target = obj
    obj.shake.shakeHandler(event)
  end)
end
```

Note the `if obj == nil` guard: these suites currently print and pass rather
than fail. Generated tests should use `assert_not_nil` instead.

Swipe direction can be synthesized without the gesture library by dispatching
the handler's expected payload straight to `obj.swipe.swipeHandler`:

```lua
local handler = M.UI.sceneGroup["rect_0"].swipe.swipeHandler
handler{phase = "ended", direction = "left", target = M.UI.sceneGroup["rect_0"]}
```

---

## 5. Environment preconditions a generated test must state

These are the failure modes a generated test hits first. Every generated suite
should assert its preconditions rather than silently pass.

1. `env.book` / `env.page` in `Solar2D/main.lua` must match the suite's
   `set(book, page)` registration and the `base_suite.new({book=, page=})`
   config. A mismatch means `M.UI.sceneGroup[name]` is `nil`.
2. `base_suite.new({ component = "iconOnly" })` skips
   `componentSelector:onClick`, so `M.layerTable.objs` stays empty. Use the
   default (`withClick`) for any test that selects layers.
3. `helper.selectLayer` needs a rendered row. `M.layerTable.objs` is populated
   by the layer store listener (`layerTable:renderLayerStore`). Suites that
   depend on it set `M.UI.testCallback` (called from
   `lua_modules/kwiksher/kwik/editor/index.lua:275` after the test run) or use
   `timer.performWithDelay`.
4. `classEntries` only exists when the layer has classes. Selecting a class
   without `classEntries` falls through and `helper.selectLayer` returns `nil`.
5. Timing-sensitive helpers (`selectIcon` 500ms, `clickIcon` 1000ms) resolve on
   timers. Assertions placed immediately after them run before the effect.
6. `M.layerTable.altDown` / `controlDown` are plain fields read at dispatch
   time; reset them in the same test so later tests are unaffected.
7. Right-click menus require `obj.isSelected`. `buttons.isLoaded` is a one-way
   toggle inside `showContextMenu`: a second call while it is still `true` runs
   `buttons:hide()` and clears the flag. `buttons:hideContextMenu()` does not
   reset it, so a test that shows two menus in a row must set
   `M.buttons.isLoaded = false` (and `M.buttons:hideContextMenu()`) between
   them, then assert `M.buttons.contextMenuOptions` after each.
8. `system.setTapDelay(0.2)` is set in `main.lua`; double-tap tests depend on it.

---

## 6. Generation template

This is the shape a coding agent should emit for one interaction on one layer.

```lua
local M = require("Test.base_suite").new({
  selectApp = true,
  book = "<book>",
  page = "<page>",
})

local helper = require("Test.helper")

local LAYER = "<layerName>"

function M.test_<interaction>_dispatch_targets_runtime()
  local obj = M.UI.sceneGroup[LAYER]
  assert_not_nil(obj, "scene object " .. LAYER .. " not found; check env.book/env.page")

  local calls = {}
  local original = obj.<handlerName>
  obj.<handlerName> = function(event)
    calls[#calls + 1] = event
    return original and original(event)
  end

  -- dispatch (see section 1 table)
  obj:touch{phase = "ended"}

  assert_len(1, calls)
  assert_equal("ended", calls[1].phase)

  obj.<handlerName> = original
end

return M
```

Rules for generated tests:

- One behavior per test function; name it `test_<layer>_<interaction>_<expected>`.
- Assert at least: the target object is non-nil, the handler ran once, and the
  observable state that changed (`UI.editor.currentLayer`, `M.UI.scene` event
  received, `obj.isSelected`, `buttons.contextMenuButtons`, layer props value).
- Restore every global you mutate (`altDown`, `controlDown`, `testCallback`,
  monkey-patched handlers) before the test returns.
- Use `assert_not_nil` with a message that names `env.book`/`env.page`, so a
  misconfigured run reads as a failure, not a silent pass.
- Prefer `helper.*` over hand-rolled loops over `M.layerTable.objs`; only drop
  to raw dispatch when the helper has no matching overload.

---

## 7. Gaps to close before auto-generation is reliable

- `helper.selectBook` has no touch fallback; it only dispatches `mouse`. Book
  selection is done via `bookTable.commandHandler(obj, {phase="ended"}, true)`.
- `helper.selectTimer` passes `groupTable` to `_selectComponent`, which looks
  like a copy-paste bug. Do not generate timer tests against it until fixed.
- There is no `createSpinEmulator` / swipe emulator, unlike pinch and shake.
- Swipe/spin/canvas suites currently have `xtest_*` placeholders only; the
  runtime dispatch shapes in section 4.4 are extracted from the layer modules,
  not from passing tests.
- `Test/index.lua` is hand-maintained; a generated suite must also add its
  `set(book, page)` line or it never runs.
