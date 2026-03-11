---
name: sketchup_api
description: Professional SketchUp Ruby API plugin development standards, idioms, and best practices. Use this skill whenever developing, reviewing, or debugging SketchUp extensions.
resources:
  - https://github.com/SketchUp/sketchup-ruby-api-tutorials
  - https://github.com/SketchUp/sketchup-safe-observer-events
  - https://github.com/SketchUp/TestUp-2
  - https://github.com/SketchUp/rubocop-sketchup
  - https://github.com/SketchUp/vscode-sketchup-extension-project
  - https://developer.sketchup.com/developers/example-extensions
---

# SketchUp Ruby API — Professional Plugin Development Guide

> **Sources:** This skill is grounded in official SketchUp GitHub repositories:
> - [`sketchup-ruby-api-tutorials`](https://github.com/SketchUp/sketchup-ruby-api-tutorials) — Official annotated examples
> - [`rubocop-sketchup`](https://github.com/SketchUp/rubocop-sketchup) — Static analysis for Extension Warehouse compliance
> - [`TestUp-2`](https://github.com/SketchUp/TestUp-2) — Minitest-based testing framework for SketchUp
> - [`sketchup-safe-observer-events`](https://github.com/SketchUp/sketchup-safe-observer-events) — Safe model changes from observers
> - [`vscode-sketchup-extension-project`](https://github.com/SketchUp/vscode-sketchup-extension-project) — Official VSCode boilerplate

You are developing a SketchUp extension in Ruby. Always follow these rules. Deviating from them produces unprofessional, fragile code that will fail on real users' machines.

---

## 1. ALWAYS Wrap Operations for Undo

Every user-visible action MUST be wrapped in `start_operation / commit_operation`. This makes the entire action undoable with a single Ctrl+Z.

```ruby
model = Sketchup.active_model
model.start_operation('My Action', true)
begin
  # all geometry / entity changes here
  model.commit_operation
rescue => e
  model.abort_operation
  raise e
end
```

- `true` as the second argument disables intermediate UI updates (performance).
- Never modify the model outside this guard.
- Never nest operations; flatten them.

---

## 2. ALWAYS Validate Geometry Before API Calls

SketchUp raises `TypeError: Zero length vector` and similar errors when you pass degenerate geometry. Guard every geometric operation:

```ruby
# ✅ Safe vector usage
vec = pt2 - pt1
return if vec.length < 0.001   # zero-length guard
vec.normalize!

# ✅ Safe edge offset
def safe_offset(edge, distance)
  vec = edge.line[1]
  return nil if vec.length < 0.001
  edge.line.offset(distance)
end

# ✅ Safe face normal
def safe_normal(face)
  return nil unless face.is_a?(Sketchup::Face)
  normal = face.normal
  return nil if normal.length < 0.001
  normal
end
```

**Never trust user selections or model geometry to be clean.** Always validate.

---

## 3. Use Observers for Reactive Behavior

Don't poll or use timers to detect model changes. Use the provided Observer API:

```ruby
# Model-level observer
class MyModelObserver < Sketchup::ModelObserver
  def onTransactionCommit(model)
    # reacts after every committed operation
    refresh_ui
  end
end

# Selection observer
class MySelectionObserver < Sketchup::SelectionObserver
  def onSelectionBulkChange(selection)
    update_toolbar_state(selection)
  end
end

# Attach in tool activation
def activate
  model = Sketchup.active_model
  @model_observer = MyModelObserver.new
  model.add_observer(@model_observer)
end

# Detach on deactivation — always clean up
def deactivate(view)
  Sketchup.active_model.remove_observer(@model_observer)
end
```

---

## 4. Extract Methods — Methods Must Be Small

Any method longer than ~25 lines MUST be split. Each method does ONE thing.

```ruby
# ❌ Bad: giant monolith
def draw(view)
  # 150 lines of font detection, geometry, drawing, caching...
end

# ✅ Good: clean separation of concerns
def draw(view)
  setup_drawing_options(view)
  draw_bounding_box(view)
  draw_label(view)
end

def setup_drawing_options(view)
  view.line_width = 2
  view.drawing_color = 'red'
end

def draw_bounding_box(view)
  pts = bounding_box_points
  view.draw(GL_LINE_LOOP, pts)
end
```

---

## 5. Never Repeat Logic — DRY

If the same logic appears in more than one method, extract it.

```ruby
# ❌ Bad: font detection in every draw method
def draw_label(view)
  font = ['Arial', 'Helvetica', 'sans-serif'].find { |f| font_available?(f) }
  # ...
end

def draw_tooltip(view)
  font = ['Arial', 'Helvetica', 'sans-serif'].find { |f| font_available?(f) }
  # ...
end

# ✅ Good: memoized, defined once
def preferred_font
  @preferred_font ||= ['Arial', 'Helvetica', 'sans-serif'].find { |f| font_available?(f) } || 'Arial'
end
```

---

## 6. Cache Expensive Results with Memoization

```ruby
# ❌ Bad: recalculates every frame
def draw(view)
  heavy_data = calculate_heavy_stuff
end

# ✅ Good: cached by object identity, invalidated on change
def entity_data(entity)
  @cache ||= {}
  @cache[entity.entityID] ||= calculate_for(entity)
end

def invalidate_cache
  @cache = nil
end
```

- Use `entity.entityID` (not `object_id`) as cache key — entity IDs are stable across Ruby GC cycles.
- Always provide an explicit invalidation path (observer callback, user action, etc.).

---

## 7. NEVER Use Temporary Files for Calculations

Exporting geometry to a file just to read its size is a critical anti-pattern. Use the API:

```ruby
# ❌ Never do this
entity.export('/tmp/tmp_calc.dae')
size = File.size('/tmp/tmp_calc.dae')
File.delete('/tmp/tmp_calc.dae')

# ✅ Use BoundingBox
bb = entity.bounds
volume_approx = bb.width * bb.height * bb.depth

# ✅ Or compute face areas directly
total_area = entity.definition.entities.grep(Sketchup::Face).sum(&:area)
```

---

## 8. Extension Structure — File Layout

```
MyPlugin/
├── main.rb              # Loader only — require all sub-files here
├── extension.rb         # Sketchup::Extension registration
├── core/
│   ├── tool.rb          # Tool class (handle user interaction)
│   ├── geometry.rb      # Pure geometry helpers (no UI)
│   └── validator.rb     # Input validation helpers
├── ui/
│   ├── toolbar.rb       # Toolbar / menu registration
│   └── dialog.rb        # HtmlDialog wrappers
└── observers/
    └── model_observer.rb
```

- `main.rb` only does `require` — no logic.
- Geometry helpers must have NO dependency on UI or `Sketchup.active_model` — pure functions.
- Validators return structured results, never raise directly.

---

## 9. Tool Class Template

```ruby
module MyPlugin
  class MyTool
    CURSOR_PENCIL = 0

    def initialize
      @mouse_position = nil
      @observer = nil
    end

    # Called when tool is activated
    def activate
      Sketchup.active_model.add_observer(setup_observer)
      update_statusbar
    end

    # Called when tool is deactivated — MUST clean up
    def deactivate(view)
      Sketchup.active_model.remove_observer(@observer)
      view.invalidate
    end

    def onMouseMove(flags, x, y, view)
      @mouse_position = view.inputpoint(x, y)
      view.invalidate
    end

    def onLButtonDown(flags, x, y, view)
      perform_action(view)
    end

    def draw(view)
      return unless @mouse_position
      draw_preview(view)
    end

    def onKeyDown(key, repeat, flags, view)
      return if repeat > 1
      perform_action(view) if key == 13   # Enter
    end

    def getStatusText
      'Click to place. Press Escape to exit.'
    end

    def getCursor
      CURSOR_PENCIL
    end

    private

    def perform_action(view)
      model = Sketchup.active_model
      model.start_operation('My Tool Action', true)
      begin
        # geometry work here
        model.commit_operation
      rescue => e
        model.abort_operation
        UI.messagebox("Error: #{e.message}")
      end
    end

    def draw_preview(view)
      # drawing code
    end

    def update_statusbar
      Sketchup.set_status_text(getStatusText)
    end

    def setup_observer
      @observer = MyPlugin::ModelObserver.new(self)
    end
  end
end
```

---

## 10. HtmlDialog (Modern UI) — Not WebDialog

```ruby
# ❌ Deprecated
dialog = UI::WebDialog.new(...)

# ✅ Modern
dialog = UI::HtmlDialog.new(
  dialog_title: 'My Plugin',
  preferences_key: 'com.mycompany.myplugin',
  width: 400,
  height: 300,
  min_width: 300,
  min_height: 200,
  resizable: true
)
dialog.set_url(File.join(__dir__, 'ui', 'index.html'))
dialog.add_action_callback('plugin_action') do |action_context, data|
  handle_ui_action(data)
end
dialog.show
```

---

## 11. Namespace Everything

NEVER pollute the global Ruby namespace.

```ruby
# ❌ Dangerous
def draw_box(entity)

# ✅ Safe
module MyCompany
  module MyPlugin
    def self.draw_box(entity)
    end
  end
end
```

Use a two-level namespace: `Company::Plugin`.

---

## 12. Numeric Tolerances

SketchUp uses inches internally. Always use `Sketchup::TOLERANCE` or define your own:

```ruby
GEOM_TOLERANCE = 1.0e-10

def points_equal?(pt1, pt2)
  pt1.distance(pt2) < GEOM_TOLERANCE
end
```

Never compare floating point lengths with `==`.

---

## 13. Safe Observer Events — Never Modify Model Directly in Callbacks

Modifying the model inside an observer callback can corrupt the undo stack or crash SketchUp.
Use deferred execution:

```ruby
# ❌ Dangerous — direct model modification inside observer
class BadObserver < Sketchup::EntitiesObserver
  def onElementAdded(entities, entity)
    entity.set_attribute('MyData', 'key', 'value')  # Can corrupt undo stack!
  end
end

# ✅ Safe — always check entity.deleted? first, defer model changes
class SafeObserver < Sketchup::EntitiesObserver
  def onElementAdded(entities, entity)
    return if entity.deleted?   # entity may already be gone
    return unless entity.is_a?(Sketchup::Face)

    # Defer model change until the current operation finishes
    UI.start_timer(0) {
      return if entity.deleted?  # double-check after deferral
      model = Sketchup.active_model
      model.start_operation('Auto-tag face', true)
      entity.set_attribute('MyPlugin', 'auto_tagged', true)
      model.commit_operation
    }
  end
end
```

**Rules for observers:**
- Always check `entity.deleted?` before accessing an entity in a callback
- Never call `start_operation` inside a callback without deferring first
- Clean up all observers in `deactivate` and `onQuit`
- Use `ModelObserver#onTransactionCommit` to rebuild caches after any operation

---

## 14. Attribute Dictionaries — Persistent Custom Data

Store custom metadata on entities using attribute dictionaries. This persists across undo/redo and file saves.

```ruby
# Write
entity.set_attribute('MyPlugin_v1', 'weight_kg', 42.5)
entity.set_attribute('MyPlugin_v1', 'is_structural', true)

# Read (with default fallback)
weight = entity.get_attribute('MyPlugin_v1', 'weight_kg', 0.0)

# Check and delete
if entity.attribute_dictionary('MyPlugin_v1')
  entity.delete_attribute('MyPlugin_v1')
end

# Iterate over all tagged entities in the model
Sketchup.active_model.entities.each do |e|
  val = e.get_attribute('MyPlugin_v1', 'weight_kg')
  next unless val
  process(e, val)
end
```

**Naming convention:** Use `'YourPluginName_v1'` as the dictionary name to avoid collisions with other extensions.

---

## 15. Testing with TestUp (Minitest inside SketchUp)

SketchUp has an official test framework: [`TestUp-2`](https://github.com/SketchUp/TestUp-2), which wraps Minitest.

```ruby
# tests/tc_my_feature.rb
require 'testup/testcase'

module MyPlugin
  module Tests
    class TC_MyFeature < TestUp::TestCase

      def setup
        @model = Sketchup.active_model
        @model.start_operation('Test Setup', true)
      end

      def teardown
        @model.abort_operation
      end

      def test_safe_offset_returns_nil_for_zero_vector
        # Test the guard clause
        result = MyPlugin::Geometry.safe_offset(nil_vec_edge, 10)
        assert_nil result
      end

      def test_entity_tagged_correctly
        face = @model.active_entities.add_face(
          [0,0,0], [10,0,0], [10,10,0], [0,10,0]
        )
        MyPlugin.tag_face(face)
        assert_equal true, face.get_attribute('MyPlugin_v1', 'is_structural')
      end

    end # class
  end # module Tests
end # module MyPlugin
```

**Testing checklist for every feature:**
```
[ ] Empty scene (no entities)
[ ] Single entity
[ ] Many entities (100+)
[ ] Degenerate geometry (zero-length edges, colinear points)
[ ] After undo / redo
[ ] With pre-selected entities
[ ] With no selection
```

---

## 16. Static Analysis — RuboCop + rubocop-sketchup

Install [`rubocop-sketchup`](https://github.com/SketchUp/rubocop-sketchup) to catch Extension Warehouse violations automatically:

```bash
gem install rubocop rubocop-sketchup
```

`.rubocop.yml` for your project:
```yaml
require: rubocop-sketchup

AllCops:
  NewCops: enable

SketchupRequirements:
  Enabled: true

SketchupSuggestions:
  Enabled: true

SketchupPerformance:
  Enabled: true
```

Run: `rubocop src/` before every commit. The `SketchupRequirements` cops enforce **mandatory** Extension Warehouse rules.

---

## 17. VSCode Project Setup (Official Boilerplate)

Use the official VSCode boilerplate: [`vscode-sketchup-extension-project`](https://github.com/SketchUp/vscode-sketchup-extension-project)

Provides:
- Ruby auto-complete with SketchUp API stubs
- Inline RuboCop linting
- Debugger configuration (attach to SketchUp process)
- Task runner for installing extension into SketchUp's Plugins folder

---

## 18. Extension Loader Pattern — The Correct Entry Point

This is how every professional SketchUp extension starts. The plugin directory contains **exactly one** loader file that follows SketchUp conventions:

```ruby
# Plugins/MyCompany_MyPlugin.rb  ← loader file (snake_case, vendor-prefixed)

require 'sketchup.rb'
require 'extensions.rb'

module MyCompany
  module MyPlugin
    unless file_loaded?(__FILE__)
      # Register the extension so it appears in Extension Manager
      ext = SketchupExtension.new('My Plugin Name', 'MyCompany_MyPlugin/main')
      ext.description = 'One clear sentence what this does.'
      ext.version     = '1.0.0'
      ext.creator     = 'Your Name'
      ext.copyright   = "#{Time.now.year}, Your Name"
      Sketchup.register_extension(ext, true)  # true = enabled by default
      file_loaded(__FILE__)
    end
  end
end
```

```ruby
# Plugins/MyCompany_MyPlugin/main.rb  ← actual plugin code starts here

module MyCompany
  module MyPlugin
    PATH = File.dirname(__FILE__).freeze

    require File.join(PATH, 'core', 'tool')
    require File.join(PATH, 'ui',   'toolbar')
    require File.join(PATH, 'ui',   'menu')
  end
end
```

**Rules:**
- Loader filename: `VendorPrefix_PluginName.rb` — no spaces, vendor-prefixed
- Use `file_loaded?` / `file_loaded` to prevent double-loading when SketchUp reloads
- Use `File.join(PATH, ...)` never string concatenation for paths
- Never use `require_relative` in SketchUp plugins — use `Sketchup.require` or `File.join`

---

## 19. Tool Class — Full Professional Lifecycle

Every callback serves a purpose. Missing any one breaks UX:

```ruby
module MyCompany
  module MyPlugin
    class DrawTool

      # ─── Cursors ───────────────────────────────────────────
      CURSOR_DEFAULT = UI.create_cursor(
        File.join(PATH, 'cursors', 'my_cursor.png'), 8, 8
      )

      def initialize
        @state        = :idle      # state machine idiom
        @picked_point = nil
        @ip           = Sketchup::InputPoint.new
        @ip_mouse     = Sketchup::InputPoint.new
      end

      # Called once when tool is pushed onto the tool stack
      def activate
        update_status_text
        Sketchup.active_model.active_view.invalidate
      end

      # Called when tool is TEMPORARILY suspended (e.g. user hits Orbit)
      def suspend(view)
        view.invalidate
      end

      # Called when tool is RESUMED after suspension
      def resume(view)
        update_status_text
        view.invalidate
      end

      # Called when tool is popped from the tool stack
      def deactivate(view)
        view.invalidate
      end

      # ─── Mouse ─────────────────────────────────────────────
      def onMouseMove(flags, x, y, view)
        @ip_mouse.pick(view, x, y, @ip)
        view.tooltip = @ip_mouse.tooltip
        view.invalidate
      end

      def onLButtonDown(flags, x, y, view)
        @ip.copy!(@ip_mouse)
        @picked_point = @ip.position
        transition_state(view)
      end

      # ─── Keyboard ──────────────────────────────────────────
      def onKeyDown(key, repeat, flags, view)
        return if repeat > 1
        case key
        when VK_ESCAPE then reset_tool(view)
        when 13        then confirm_action(view)      # Enter
        end
        false  # return false to allow SketchUp to handle key too
      end

      # ─── Drawing ───────────────────────────────────────────
      # draw is called EVERY frame. Never compute here — only render cached data.
      def draw(view)
        draw_inference_lines(view)
        draw_preview_geometry(view)
        @ip_mouse.draw(view) if @ip_mouse.valid?
      end

      # MUST return a BoundingBox that covers ALL custom geometry you draw.
      # Without this, SketchUp clips your graphics when the camera moves.
      def getExtents
        bb = Geom::BoundingBox.new
        bb.add(@picked_point) if @picked_point
        bb.add(@ip_mouse.position) if @ip_mouse.valid?
        bb
      end

      # Called very frequently — keep it microsecond-fast
      def onSetCursor
        UI.set_cursor(CURSOR_DEFAULT)
      end

      def getInstructionText
        'Pick first point'
      end

      def getStatusText
        case @state
        when :idle    then 'Click to start'
        when :drawing then 'Click to finish, Escape to cancel'
        end
      end

      private

      def transition_state(view)
        @state = @state == :idle ? :drawing : :idle
        update_status_text
        perform_action if @state == :idle
      end

      def update_status_text
        Sketchup.set_status_text(getStatusText, SB_PROMPT)
        Sketchup.set_status_text('', SB_VCB_LABEL)
      end

      def reset_tool(view)
        @state = :idle
        @picked_point = nil
        update_status_text
        view.invalidate
      end

      def draw_inference_lines(view)
        return unless @ip.valid? && @ip_mouse.valid?
        view.line_stipple = '_'
        view.line_width   = 1
        view.drawing_color = Sketchup::Color.new(128, 128, 128)
        view.draw_line(@ip.position, @ip_mouse.position)
      end

      def draw_preview_geometry(view)
        return unless @picked_point && @ip_mouse.valid?
        view.line_width    = 2
        view.drawing_color = Sketchup::Color.new(255, 0, 0)
        view.draw(GL_LINES, [@picked_point, @ip_mouse.position])
      end

      def perform_action
        model = Sketchup.active_model
        model.start_operation('Draw Element', true)
        begin
          # geometry creation here
          model.commit_operation
        rescue => e
          model.abort_operation
          UI.messagebox("Error: #{e.message}")
        end
      end

    end
  end
end
```

---

## 20. Entity Traversal at Scale — Don't Iterate Naively

Recursive traversal of large models is the #1 performance killer. Use the right strategies:

```ruby
# ❌ Never: naive nested iteration (O(n²))
model.entities.each do |e|
  if e.is_a?(Sketchup::Group)
    e.entities.each do |child|
      # ...
    end
  end
end

# ✅ Pattern 1: model.active_entities for current context only
model.active_entities.grep(Sketchup::Face).each do |face|
  process(face)
end

# ✅ Pattern 2: recursive traversal with depth guard
def traverse(entities, depth = 0, max_depth = 5, &block)
  return if depth > max_depth
  entities.each do |e|
    block.call(e)
    case e
    when Sketchup::Group
      traverse(e.entities, depth + 1, max_depth, &block)
    when Sketchup::ComponentInstance
      traverse(e.definition.entities, depth + 1, max_depth, &block)
    end
  end
end

# ✅ Pattern 3: use ObjectSpace for model-wide entity access (expert)
# Collect to array FIRST, then process — never modify during iteration
faces = []
model.active_entities.each { |e| faces << e if e.is_a?(Sketchup::Face) }
faces.each { |f| process(f) }    # safe: processing a separate array

# ✅ Pattern 4: Set for O(1) lookups instead of Array#include?
processed_ids = Set.new
model.active_entities.each do |e|
  next if processed_ids.include?(e.entityID)
  processed_ids.add(e.entityID)
  process(e)
end
```

**Performance rules (from SketchUcation community):**
- Prefer `Hash` / `Set` over `Array` for any lookup, membership test, or uniqueness tracking
- Collect entities to an array before modifying — never add/remove during `.each`
- Prefer native API methods over Ruby implementations (SketchUp API is C/C++ underneath)
- Minimize calls inside tight loops: cache `model`, `entities`, `selection` references before the loop

---

## 21. Cross-Platform Path Handling (Mac + Windows)

Plugins MUST work on both macOS and Windows. Path handling is the most common failure point:

```ruby
# ❌ Never use backslashes or platform-specific separators
path = "C:\\Users\\foo\\Plugins\\MyPlugin\\icon.png"  # Windows only

# ❌ Never use string concatenation for paths
path = __dir__ + '/icons/' + 'my_icon.png'

# ✅ Always use File.join — automatically uses correct separator
PLUGIN_PATH  = File.dirname(__FILE__).freeze
ICONS_PATH   = File.join(PLUGIN_PATH, 'icons').freeze
ICON_MAIN    = File.join(ICONS_PATH,  'main_24.png').freeze

# ✅ Encoding: force UTF-8 when reading/writing any file
content = File.read(path, encoding: 'UTF-8')

# ✅ Temp files (when unavoidable): use OS temp dir
tmp = File.join(Dir.tmpdir, "myplugin_#{Process.pid}.tmp")
begin
  # use tmp
ensure
  File.delete(tmp) if File.exist?(tmp)
end
```

**Mac-specific gotchas:**
- Toolbar icons on Mac must be `24x24` PNG, retina variants at `48x48` (named `icon_24@2x.png`)
- `UI.messagebox` blocks on Mac — never call inside an observer callback
- File paths from SketchUp dialogs on Mac return POSIX paths — never assume Windows format

---

## 22. Toolbar & Menu — Extension Warehouse UX Rules

```ruby
module MyCompany
  module MyPlugin
    unless file_loaded?(__FILE__ + 'ui')

      # ── Menus ──────────────────────────────────────────────
      # Every toolbar command MUST also be in the menu (for keyboard shortcuts)
      plugins_menu = UI.menu('Plugins')
      my_menu      = plugins_menu.add_submenu('My Plugin')

      cmd_draw = UI::Command.new('Draw Element') {
        Sketchup.active_model.select_tool(DrawTool.new)
      }
      cmd_draw.tooltip      = 'Draw a new element'
      cmd_draw.status_bar_text = 'Click to activate Draw Element tool'
      # Icon sizes: 16x16 (small toolbar), 24x24 (large toolbar)
      cmd_draw.small_icon   = File.join(ICONS_PATH, 'draw_16.png')
      cmd_draw.large_icon   = File.join(ICONS_PATH, 'draw_24.png')

      my_menu.add_item(cmd_draw)

      # ── Toolbar ────────────────────────────────────────────
      toolbar = UI::Toolbar.new('My Plugin')
      toolbar.add_item(cmd_draw)
      toolbar.restore  # remember user's show/hide preference

      file_loaded(__FILE__ + 'ui')
    end
  end
end
```

**Extension Warehouse UX rules:**
- Toolbar must be minimal — users have many extensions
- Every toolbar button MUST have a menu equivalent (for shortcut assignment)
- `cmd.tooltip` is shown on hover; `cmd.status_bar_text` is shown in the status bar — both are required
- Use `toolbar.restore` to remember visibility between sessions
- Icon sizes: 16×16 for small toolbar, 24×24 for large toolbar; PNG with transparency
- Prefer submenu under `Plugins` menu — never add top-level menus

---

## 23. SketchUp Version Guards

Your plugin may be installed on older SketchUp versions. Always guard version-specific API calls:

```ruby
# Check at load time — warn and abort gracefully
unless Sketchup.version.to_f >= 2020.0
  UI.messagebox(
    'My Plugin requires SketchUp 2020 or later.',
    MB_OK, 'My Plugin'
  )
  # Don't load the rest of the plugin
  return
end

# Guard specific API features inline
if Sketchup.version.to_f >= 2021.0
  # Use newer API
  model.active_layer_collection
else
  # Fall back to older API
  model.layers
end

# Use respond_to? for really safe feature detection
if entity.respond_to?(:definition)
  defn = entity.definition
end
```

---

## 24. Error Handling — Never Let the Plugin Crash SketchUp

```ruby
# ✅ Top-level rescue in every public entry point
def self.run_feature
  model = Sketchup.active_model
  model.start_operation('My Feature', true)
  begin
    do_the_work(model)
    model.commit_operation
  rescue Sketchup::Error => e
    # SketchUp-specific error (invalid geometry, locked layer, etc.)
    model.abort_operation
    UI.messagebox("SketchUp error: #{e.message}\n\nThe operation was cancelled.", MB_OK)
  rescue ArgumentError, TypeError => e
    # Our own logic errors — these should NEVER reach the user
    model.abort_operation
    UI.messagebox("Internal error: #{e.message}\n\nPlease report this.", MB_OK)
    raise e  # re-raise for the Ruby console in dev builds
  rescue => e
    model.abort_operation
    raise e  # unknown errors always re-raise
  end
end

# ✅ Defensive selection access
def safe_first_face(selection)
  selection.find { |e| e.is_a?(Sketchup::Face) && !e.deleted? }
end
```

---

## 25. Anti-Patterns — What Professional SketchUp Developers NEVER Do

These patterns are the hallmarks of LLM-generated or amateur plugin code:

| Anti-pattern | Correct Approach |
|---|---|
| `entity.export('/tmp/x.dae')` to measure size | Use `entity.bounds` or face areas |
| Font detection copied in every `draw` method | Extract to `memoized_font` helper |
| `sleep(0.1)` to wait for SketchUp to update | Use `view.invalidate` + observer pattern |
| `loop do … end` polling for state changes | Use `Sketchup::ModelObserver` |
| `puts` for debugging in production code | Use `Sketchup.status_text` or remove entirely |
| `rescue nil` to silence errors | Always handle errors explicitly |
| `require_relative` in plugin files | Always use `Sketchup.require` or `File.join` |
| Storing entity references across operations | Validate `entity.deleted?` every time |
| Giant 200-line `initialize` methods | Extract to `setup_*` private methods |
| `model.active_entities.to_a.each` for huge models | Use lazy iteration, `grep`, depth limits |
| `UI::WebDialog` | Always use `UI::HtmlDialog` |
| `$global_variables` | Always use `Module::CONSTANT` or `@instance_vars` |
| Modifying model inside observer without deferral | Use `UI.start_timer(0)` for deferral |
| Hardcoded `"C:/Users/..."` paths | Use `__dir__`, `File.join`, `Dir.tmpdir` |

---

## 26. Extension Packaging — .rbz and Deployment

```
MyCompany_MyPlugin/           ← matches loader name
├── MyCompany_MyPlugin.rb     ← loader (goes in Plugins root)
└── MyCompany_MyPlugin/
    ├── main.rb
    ├── core/
    ├── ui/
    │   └── index.html
    ├── icons/
    │   ├── draw_16.png
    │   └── draw_24.png
    └── CHANGELOG.md
```

**Package as .rbz (ZIP renamed):**
```powershell
# Windows PowerShell
Compress-Archive -Path .\MyCompany_MyPlugin.rb, .\MyCompany_MyPlugin\ `
                 -DestinationPath .\MyCompany_MyPlugin-1.0.0.zip
Rename-Item .\MyCompany_MyPlugin-1.0.0.zip MyCompany_MyPlugin-1.0.0.rbz
```

**Extension Warehouse minimum requirements checklist:**
```
[ ] Unique vendor-prefixed filename (VendorName_PluginName.rb)
[ ] SketchupExtension registered with name, description, version, creator
[ ] Works on Mac + Windows (tested both)
[ ] No crashes on empty scene
[ ] No modification of global Ruby namespace ($vars, top-level defs)
[ ] Graceful error messages — no bare Ruby stack traces shown to user
[ ] Compatible with at least last 3 SketchUp versions (version guards in place)
[ ] No external network calls without user consent
[ ] Icons at both 16×16 and 24×24 sizes
[ ] rubocop-sketchup SketchupRequirements passes with zero violations
```

---

## MASTER CHECKLIST — Before Every Plugin Release

### Code Quality
```
[ ] Every method < 25 lines
[ ] No repeated logic anywhere (DRY)
[ ] All namespace inside two-level Module::Module
[ ] No global variables ($var) or top-level method definitions
[ ] rubocop-sketchup passes with zero violations
```

### Safety & Correctness
```
[ ] Every model change wrapped in start_operation / commit_operation / rescue abort
[ ] Every vector/normal validated for zero length before use
[ ] entity.deleted? checked before every entity access in observers
[ ] No file I/O used for geometric calculations
[ ] Version guard at loader level (minimum SketchUp version check)
```

### Performance
```
[ ] draw method contains ZERO calculations — only renders cached data
[ ] getExtents returns accurate BoundingBox covering all drawn geometry
[ ] onSetCursor does ZERO work beyond UI.set_cursor(ID)
[ ] Expensive traversals use Set/Hash for lookups, not Array#include?
[ ] Observers cleaned up in deactivate — no memory leaks
```

### UX & Packaging
```
[ ] Every toolbar command also accessible via menu (for shortcut assignment)
[ ] toolbar.restore called so visibility preference is remembered
[ ] Both small_icon (16px) and large_icon (24px) set on every UI::Command
[ ] Status bar text is clear and context-sensitive (SB_PROMPT)
[ ] Tested on: empty scene, 1 entity, 100+ entities, degenerate geometry
[ ] Tested: undo/redo cycle, suspend/resume tool, Mac + Windows paths
[ ] .rbz package installs cleanly via Extension Manager
```


---

## 27. VCB — Measurement Bar Input (onUserText / enableVCB?)

The **VCB** (Value Control Box) is the measurement box at the bottom-right of SketchUp. Native tools use it constantly. Your tools must too.

> Source: `examples/04_vcb_tool` — copyright Trimble Inc, MIT License

```ruby
class LineTool

  def onUserText(text, view)
    begin
      distance = text.to_l   # parses "1m", "3'6\"", "50cm" — respects model units
    rescue ArgumentError
      UI.messagebox('Invalid length')
      return
    end
    direction = @mouse_ip.position - @picked_first_ip.position
    end_point = @picked_first_ip.position.offset(direction, distance)
    @mouse_ip = Sketchup::InputPoint.new(end_point)
    create_edge > 0 ? reset_tool : @picked_first_ip.copy!(@mouse_ip)
    view.invalidate
  end

  def enableVCB?
    @picked_first_ip.valid?   # true = measurement bar accepts input
  end

  private

  def update_ui
    if @picked_first_ip.valid?
      Sketchup.status_text = 'Select end point.'
      Sketchup.vcb_value = @mouse_ip.position.distance(@picked_first_ip.position)  # Length, not Float
    else
      Sketchup.status_text = 'Select start point.'
    end
    Sketchup.vcb_label = 'Length'
  end
end
```

**VCB rules:**
- Always implement `enableVCB?` — without it keypresses bypass VCB and go to SketchUp shortcuts
- `text.to_l` parses all unit formats — never parse lengths manually
- Always `rescue ArgumentError` from `to_l`
- Set `vcb_label` in `update_ui`, not in `onUserText`

---

## 28. Press–Drag–Release — Distinguish Click from Drag

> Source: `examples/03_press_drag_release_tool` — copyright Trimble Inc, MIT License

```ruby
class LineTool

  DRAG_THRESHOLD = 10   # logical screen pixels — same value native SketchUp tools use

  def onLButtonDown(flags, x, y, view)
    @mouse_down = Geom::Point3d.new(x, y)
    # ... handle first-point click logic
  end

  def onLButtonUp(flags, x, y, view)
    if @mouse_down.distance([x, y]) > DRAG_THRESHOLD
      create_edge
      reset_tool   # drag = one-shot gesture, always resets
    end
    # barely moved = click = already handled by onLButtonDown
  end
end
```

---

## 29. onCancel — Escape Must Reset the Tool

> Source: `examples/02_custom_tool` — copyright Trimble Inc, MIT License

```ruby
# reason: 0 = Escape, 1 = Reactivate, 2 = Undo
def onCancel(reason, view)
  reset_tool
  view.invalidate
end

def reset_tool
  @picked_first_ip.clear   # use .clear, never = nil, on InputPoint
  @lock_helper&.unlock
  update_ui
end
```

---

## 30. Inference Locking — Shift + Arrow Keys (Native Quality Must-Have)

> Source: `examples/05_tool_inference_lock/inference_lock_helper.rb` — copyright Trimble Inc, MIT License

```ruby
module MyPlugin
  class InferenceLockHelper

    def initialize
      @axis_lock = nil
    end

    def on_keydown(key, view, active_ip, reference_ip = active_ip)
      if key == CONSTRAIN_MODIFIER_KEY   # Shift
        try_lock_constraint(view, active_ip)
      else
        try_lock_axis(key, view, reference_ip)
      end
    end

    def on_keyup(key, view)
      return unless key == CONSTRAIN_MODIFIER_KEY
      return if @axis_lock
      view.lock_inference   # no args = unlock
    end

    def unlock
      @axis_lock = nil
      Sketchup.active_model.active_view.lock_inference
    end

    private

    def try_lock_constraint(view, active_ip)
      return if @axis_lock
      return unless active_ip.valid?
      view.lock_inference(active_ip)
    end

    def try_lock_axis(key, view, reference_ip)
      return unless reference_ip.valid?
      axes = view.model.axes
      case key
      when VK_RIGHT then lock_axis([reference_ip.position, axes.xaxis], view)
      when VK_LEFT  then lock_axis([reference_ip.position, axes.yaxis], view)
      when VK_UP    then lock_axis([reference_ip.position, axes.zaxis], view)
      end
    end

    def lock_axis(line, view)
      if line == @axis_lock
        @axis_lock = nil
        view.lock_inference
      else
        @axis_lock = line
        view.lock_inference(
          Sketchup::InputPoint.new(line[0]),
          Sketchup::InputPoint.new(line[0].offset(line[1]))
        )
      end
    end
  end
end
```

**Wire into your Tool class:**

```ruby
def activate
  @ip          = Sketchup::InputPoint.new
  @ip_mouse    = Sketchup::InputPoint.new
  @lock_helper = MyPlugin::InferenceLockHelper.new
  @mouse_pos   = ORIGIN
end

def deactivate(view)
  @lock_helper.unlock
  view.invalidate
end

def onKeyDown(key, _repeat, _flags, view)
  @lock_helper.on_keydown(key, view, @ip_mouse, @ip)
  pick_mouse_position(view)   # ALWAYS re-pick after lock change
  false   # let SketchUp handle the key too
end

def onKeyUp(key, _repeat, _flags, view)
  @lock_helper.on_keyup(key, view)
  pick_mouse_position(view)
  false
end

def onMouseMove(flags, x, y, view)
  @mouse_pos = Geom::Point3d.new(x, y, 0)   # save for re-pick on key events
  pick_mouse_position(view)
end

private

def pick_mouse_position(view)
  if @ip.valid?
    @ip_mouse.pick(view, @mouse_pos.x, @mouse_pos.y, @ip)
  else
    @ip_mouse.pick(view, @mouse_pos.x, @mouse_pos.y)
  end
  view.tooltip = @ip_mouse.tooltip if @ip_mouse.valid?
  view.invalidate
end

def draw_preview(view)
  return unless @ip.valid? && @ip_mouse.valid?
  points = [@ip.position, @ip_mouse.position]
  view.line_width = view.inference_locked? ? 3 : 1   # thicker = locked
  view.set_color_from_line(*points)   # red/green/blue by axis — never hardcode colors
  view.draw(GL_LINES, points)
end
```

**Key details (from Trimble source):**
- Always re-pick InputPoint after lock change — the point does not auto-update
- `view.inference_locked?` — use to show thicker line when locked (same as native tools)
- `view.set_color_from_line(pt1, pt2)` — colors line by snapped axis; never hardcode line colors
- `return false` from `onKeyDown/Up` so SketchUp processes the key too
- Pressing the same arrow key twice toggles the lock off


---

## 31. Known API Bugs & Workarounds — Issue Tracker Findings

> Source: [SketchUp/api-issue-tracker](https://github.com/SketchUp/api-issue-tracker) — official public bug tracker

These are confirmed, logged bugs that affect real plugins. Know them so you can work around them.

---

### BUG: AppObserver.onNewModel — UI calls cause bugsplat (SU 2024+)
**Issue:** [#1006](https://github.com/SketchUp/api-issue-tracker/issues/1006)
Calling `puts`, `UI.messagebox`, or any UI interaction inside `AppObserver.onNewModel` causes intermittent bugsplat after File → New.

```ruby
# ❌ Crashes SU2024 intermittently
class MyAppObserver < Sketchup::AppObserver
  def onNewModel(model)
    UI.messagebox('New model opened')   # bugsplat!
  end
end

# ✅ Workaround: defer with UI.start_timer(0)
class MyAppObserver < Sketchup::AppObserver
  def onNewModel(model)
    UI.start_timer(0, false) {
      UI.messagebox('New model opened')   # safe
    }
  end
end
```

**General rule:** ANY UI interaction inside an AppObserver callback must be deferred with `UI.start_timer(0, false)`.

---

### BUG: Face#area returns wrong value with glued components (SU 2023–2024)
**Issue:** [#964](https://github.com/SketchUp/api-issue-tracker/issues/964)
`Face#area` reports incorrect area when there are glued components on the face's outer loop. Regression introduced in SU 2023.1.

```ruby
# ❌ May return wrong area in SU 2023-2024 if glued components exist on the face
area = face.area

# ✅ Workaround: check by computing from mesh vertices instead
def mesh_area(face)
  mesh = face.mesh
  total = 0.0
  (1..mesh.count_polygons).each do |i|
    pts = mesh.polygon_points_at(i)
    next unless pts.size >= 3
    # Triangle area via cross product
    v1 = pts[1] - pts[0]
    v2 = pts[2] - pts[0]
    total += v1.cross(v2).length / 2.0
  end
  total
end
```

**Defensive rule:** When accurate area calculation is critical and glued components may be present, use mesh-based calculation.

---

### BUG: Enter key not triggering in Tool.onKeyDown (SU 2026)
**Issue:** [#1076](https://github.com/SketchUp/api-issue-tracker/issues/1076)
In SketchUp 2026, pressing Enter inside a custom tool does not always trigger `onKeyDown` / `onKeyUp`.

```ruby
# ❌ Unreliable in SU 2026: key == 13
def onKeyDown(key, repeat, flags, view)
  perform_action if key == 13
end

# ✅ Workaround: also handle via onReturn if available, and use VCB confirmation
def onUserText(text, view)
  # User can type a value and press Enter → triggers onUserText reliably
  confirm_action(text, view)
end
```

**Defensive rule:** For Enter-to-confirm actions, always also support VCB input (`onUserText`) as a reliable alternative.

---

### BUG: Material#name= does not guarantee unique names (SU 2024+)
**Issue:** [#1077](https://github.com/SketchUp/api-issue-tracker/issues/1077)
Setting `material.name =` does not sanitize or enforce uniqueness — can result in duplicate material names in the model.

```ruby
# ❌ May create duplicate names
mat = model.materials.add('Wood')
mat.name = 'Concrete'   # no uniqueness check

# ✅ Check for uniqueness yourself
def unique_material_name(model, base_name)
  existing = model.materials.map(&:name)
  return base_name unless existing.include?(base_name)
  i = 1
  i += 1 while existing.include?("#{base_name}_#{i}")
  "#{base_name}_#{i}"
end

mat = model.materials.add(unique_material_name(model, 'Concrete'))
```

---

### BUG: HtmlDialog fetch requests hang randomly (SU 2024)
**Issue:** [#991](https://github.com/SketchUp/api-issue-tracker/issues/991)
`fetch()` calls from inside an `HtmlDialog` page hang indefinitely in SU 2024 on some systems.

```ruby
# ✅ Workaround: use add_action_callback for all data exchange instead of fetch
dialog.add_action_callback('get_data') do |context, key|
  # Return data via sketchup.callbackName
  dialog.execute_script("receiveData(#{data.to_json})")
end
```

**Rule:** Never rely on `fetch()` for SketchUp ↔ dialog communication. Always use `add_action_callback` + `execute_script`.

---

### BUG: Non-unique toolbar icon images not shown (Open issue)
**Issue:** [#1080](https://github.com/SketchUp/api-issue-tracker/issues/1080)
If two `UI::Command` objects reference the same icon path string (even across different plugins), only one icon shows.

```ruby
# ✅ Always use absolute, unique paths per command
cmd.large_icon = File.join(ICONS_PATH, 'my_unique_draw_24.png')
# Ensure the actual PNG filenames are unique across your plugin
```

---

### KNOWN LIMITATION: Undo stack not exposed via API
The undo/redo stack contents and operation names are not accessible from Ruby. You cannot query "what was the last undone operation" in observer callbacks.

**Workaround:** Maintain your own state cache and compare against the model in `ModelObserver#onTransactionUndo`:

```ruby
class MyModelObserver < Sketchup::ModelObserver
  def onTransactionUndo(model)
    # Cannot read what was undone — must re-scan model
    rebuild_cache_from_model(model)
  end

  def onTransactionRedo(model)
    rebuild_cache_from_model(model)
  end
end
```

---

### Tracker Quick Reference

| Issue | Versions | Status | Workaround |
|-------|---------|--------|------------|
| `AppObserver.onNewModel` UI bugsplat | SU 2024+ | Closed/logged | `UI.start_timer(0)` deferral |
| `Face#area` wrong with glued components | SU 2023–2024 | Closed/logged | Mesh-based area calculation |
| Enter key unreliable in Tool | SU 2026 | Open | Use `onUserText` as fallback |
| `Material#name=` no uniqueness check | SU 2024+ | Open | Manual uniqueness function |
| `HtmlDialog` fetch hangs | SU 2024 | Closed | `add_action_callback` only |
| Duplicate icon paths not shown | All | Open | Unique filenames per command |



---

## 32. TestUp-2 — Writing Real Tests Inside SketchUp (Trimble Patterns)

> Source: [SketchUp/testup-2](https://github.com/SketchUp/testup-2) — © Trimble Inc, MIT License
> The full official test suite is at `tests/SketchUp Ruby API/` — 70+ TC_ files covering every class.

---

### Setup and Installation

```ruby
# In Ruby Console — install minitest gem if getting gem errors:
Gem.install('/path/to/testup-2/gems/minitest-5.15.0.gem')
# Then restart SketchUp.

# load_testup.rb — put in Plugins folder to load from source:
$LOAD_PATH << "C:/Users/YourName/testup-2/src"
require "testup.rb"
```

---

### Test File Structure (from official TC_ files)

```ruby
# File: TC_MyPlugin_MyClass.rb
# Convention: TC_ prefix, then module + class name

require "testup/testcase"
require_relative "utils/version_helper"   # optional — for skip helpers

class TC_MyPlugin_MyClass < TestUp::TestCase

  include TestUp::SketchUpTests::VersionHelper   # adds sketchup_older_than()

  # Called ONCE before all tests in this class — use to open test models
  def self.setup_testcase
    discard_all_models
  end

  # Called before EACH test — always start clean
  def setup
    start_with_empty_model
  end

  def teardown
    # cleanup if needed
  end


  # ======================================================================== #
  # Factory helpers — shared setup logic extracted into methods

  def create_face
    entities = Sketchup.active_model.active_entities
    face = entities.add_face([0,0,0], [100,0,0], [100,100,0], [0,100,0])
    face.reverse!
    face
  end

  def create_test_material(filename)
    path = get_test_case_file(filename)
    mat = Sketchup.active_model.materials.add("Test Material")
    mat.texture = path
    mat
  end

  def get_test_case_file(filename)
    File.join(__dir__, File.basename(__FILE__, '.*'), filename)
  end


  # ======================================================================== #
  # Test method naming convention:
  #   test_<method_name>_<scenario>
  # Group by method using separator comments

  # ======================================================================== #
  # method MyClass#my_method

  # 1. Smoke test — run the API example
  def test_my_method_api_example
    face = create_face
    result = face.my_method
    # just assert no exception thrown
  end

  # 2. Return type
  def test_my_method_return_type
    face = create_face
    assert_kind_of(Float, face.my_method)
  end

  # 3. Return value
  def test_my_method_return_value
    face = create_face
    # Use SKETCHUP_FLOAT_TOLERANCE for geometric comparisons — never ==
    assert_in_delta(10000.0, face.my_method, SKETCHUP_FLOAT_TOLERANCE)
  end

  # 4. Arity check — ensure method signature hasn't changed across SU versions
  def test_my_method_arity
    assert_equal(0, MyClass.instance_method(:my_method).arity)
  end

  # 5. Invalid arguments
  def test_my_method_incorrect_arguments
    face = create_face
    assert_raises(ArgumentError) { face.my_method("bad") }
    assert_raises(ArgumentError) { face.my_method(nil) }
    assert_raises(TypeError)     { face.my_method(false) }
  end

  # 6. Version guard — skip test on old versions
  def test_my_method_new_feature
    skip('Added in SU2021.1') if Sketchup.version.to_f < 21.1
    # test new API...
  end

end
```

---

### Key Assertions and Helpers

```ruby
# Float / geometric comparisons — NEVER use == for floats
assert_in_delta(expected, actual, SKETCHUP_FLOAT_TOLERANCE)

# Type checks
assert_kind_of(Sketchup::Face, result)
assert_kind_of(Array, result)
assert_nil(result)

# Exception testing — per-exception-type, per-bad-input
assert_raises(ArgumentError) { face.method(nil) }
assert_raises(TypeError)     { face.method("bad") }

# Skip tests conditionally
skip('Fixed in SU2019.2') if Sketchup.version.to_f < 19.2
skip('Added in SU2021.1') if Sketchup.version.to_f < 21.1
skip('Broken in SU2014')   if Sketchup.version.to_i == 14

# Version helper module (from utils/version_helper.rb)
include TestUp::SketchUpTests::VersionHelper
skip('...') if sketchup_older_than(2021, 1)   # cleaner than Sketchup.version.to_f

# Arity: -1 means optional args, 0 means no args, N means exactly N
assert_equal(-1, Sketchup::Face.instance_method(:area).arity)
assert_equal(0,  Sketchup::Face.instance_method(:edges).arity)

# Collection filtering (idiomatic Ruby in tests)
faces = connected.grep(Sketchup::Face)
assert_equal(1, faces.size)

# Entity validity
assert(connected.all? { |e| e.valid? })
assert(entities.all? { |e| e.parent == model })
```

---

### Running Tests from Terminal (CI Integration)

```sh
# Windows — run a full test suite path:
"C:\Program Files\SketchUp\SketchUp 2026\SketchUp.exe" ^
  -RubyStartupArg "TestUp:CI:Path: C:\my_plugin\tests" > results.json

# Windows — run with YAML config:
"C:\Program Files\SketchUp\SketchUp 2026\SketchUp.exe" ^
  -RubyStartupArg "TestUp:CI:Config: C:\my_plugin\testup-ci.yml"
```

```yaml
# testup-ci.yml
Path: "%CONFIG_DIR%/tests"         # %CONFIG_DIR% = dir where YAML lives
Output: "%CONFIG_DIR%/results.json"
Seed: 123                          # fixed seed for deterministic ordering
KeepOpen: false                    # SketchUp closes after tests
Tests:                             # run only specific tests
  - TC_MyClass#                    # entire test case
  - TC_MyClass#test_specific_case  # single test
```

```ruby
# From Ruby Console:
path = "C:/my_plugin/tests"
TestUp::API.run_tests(["TC_MyClass#test_specific"], path: path, options: { ui: false })
```

---

### What Trimble Tests for Every Method (Checklist)

The official TC_ files are a gold standard. For every API method they test:

| Test | What it checks |
|------|---------------|
| `_api_example` | The API docs example runs without error |
| `_return_type` | `assert_kind_of` on return value class |
| `_return_value` | `assert_in_delta` / `assert_equal` on actual values |
| `_arity` | `instance_method(:name).arity` hasn't changed |
| `_incorrect_number_of_arguments` | Raises on too few/many args |
| `_invalid_arguments` | Raises correct exception type per bad input |
| Version-guarded tests | `skip(...)` with exact version where feature was added |

**Apply the same checklist to your own plugin methods.**

---

### Rubocop Configuration (from official .rubocop.yml)

```yaml
require:
  - rubocop-performance
  - rubocop-sketchup

AllCops:
  SketchUp:
    SourcePath: src                  # only lint plugin source, not tests
    TargetSketchUpVersion: 2017      # enables version-aware cops
  Exclude:
    - tests/**/*.rb                  # tests use different naming conventions

# Minimum required cops:
SketchupDeprecations: { Enabled: true }
SketchupPerformance:  { Enabled: true }
SketchupRequirements: { Enabled: true }
SketchupSuggestions:  { Enabled: true }
SketchupBugs:         { Enabled: true }   # catches API-specific bugs

# Tool classes must use camelCase SU API method names — exclude from Naming cop:
Naming/MethodName:
  Exclude:
    - '**/*_tool.rb'

# x, y, z, u, v, r, g, b are acceptable short names in 3D context:
Naming/MethodParameterName:
  Enabled: false
```

---

## 33. Extension UX Guidelines — General Principles

> Source: [SketchUp/sketchup-extension-ux-guidelines](https://github.com/SketchUp/sketchup-extension-ux-guidelines)

**Language and Tone:** User-facing text should be friendly and non-technical. Avoid exclamation marks in error messages (e.g., `Trial expired` instead of `TRIAL EXPIRED!`).

**Respect the Undo Stack:**
- **One user action = one undo step.** Wrap multiple model changes from a single user action into one `start_operation` / `commit_operation`.
- Do not silently modify the model on load or when UI is shown. Focus changes strictly on what the user initiated.

**Honor Settings & State:**
- **Locked Entities:** Do not modify locked entities unless your explicit purpose is to unlock them. Always filter the selection:
  ```ruby
  # ✅ Good
  selection.reject(&:locked?).each { |e| process(e) }
  ```
- **Group Uniqueness:** When editing a group via API, silently make it unique first (`group.make_unique`) to mimic SketchUp's native behavior, preventing accidental modifications to copies.
- **Lengths:** Always use SketchUp's native length parsing (`.to_l`) and formatting to respect the user's model units.

---

## 34. Extension UX Guidelines — Menus and Toolbars

**Menus:**
- Prefer the **Extensions** menu.
- Always group your commands in a **Submenu** named after your extension.
- **Never** add separators to the Extensions menu. For other menus, add a separator *before* your extension, but never after.

**Toolbars:**
- Optimize space: combine related toggles into single buttons. Dispense with rarely used commands (Help/About) by leaving them only in the menu.
- Do not include version numbers in command titles or toolbar names. It breaks user shortcuts and toolbar layouts upon updates.
- **Tooltip vs Status Bar:**
  - `tooltip` (short text) pops up on hover (e.g., `Rectangle Tool`).
  - `status_bar_text` (full sentence) explains what the command does, starting with an action verb (e.g., `Draw a rectangle from corner to corner.`).

---

## 35. Extension UX Guidelines — Interactive Tools

If your plugin implements the `Tool` interface:

- **Esc Resets the Tool:** Pressing Esc should back out of the current tool's state (e.g. clear `InputPoint`s) but should **not** switch to the Select tool. Stay in your tool.
- **Pre-Selection vs In-Tool Selection:**
  - If a tool requires a selection, let the user pick objects *after* activating the tool instead of showing an error.
  - If objects are already selected (pre-selection) before activating the tool, skip the selection stage to save clicks.
- **Interaction Styles:** Support both `Click + Move + Click` (ergonomic, wrist-friendly) and `Press + Drag + Release`.
- **Scale Agnosticism:** Visuals (like protractors) and interaction speeds should feel consistent regardless of zoom level or model size.
- **Visual Cues:** Use the Status Bar to explain modifier keys clearly, and convey 3D depth by drawing 3D handles instead of flat 2D screen handles.

---

## 36. Extension UX Guidelines — Dialogs and Presentation

**HtmlDialogs:**
- **Link Dialog to Selection:** If a dialog edits properties, make sure it stays synchronized when the user's selection changes by using a `SelectionObserver`.
- **Modus UI Framework:** If you don't have a strong brand identity, use [Modus for SketchUp Extensions](https://sketchup.github.io/modus-for-sketchup-extensions/), SketchUp's standard UI framework.

**Extension Presentation (Extension Warehouse):**
- **What before How:** Start with *what* the extension does and *why* it's useful. Keep the instructions on *how* to use it minimal or at the bottom.
- Be courteous and specifically mention where the extension can be found in the SketchUp UI (e.g. "Available in Extensions > My Plugin").
- Avoid excessive formatting (too many headers/bold text) that makes scanning difficult.



---

## 37. Strict API Namespace & Global Rules (RuboCop Cops)

To comply with Extension Warehouse and prevent clashes with other plugins:
- **No Global Variables/Constants/Methods**: Everything must be inside your Company::Plugin namespace. Do not use $my_global.
- **Never Modify SketchUp API or Ruby Core/StdLib**: Do not monkey-patch Sketchup classes (like Sketchup::Model), String, Array, etc.
- **Isolate Dependencies**: Do not use Gem.install at runtime. Namespace your dependencies and bundle them with your extension.
- **Do Not Change Global Debug Mode**: Leave Sketchup.debug_mode= alone in production.
- **Never Rescue Exception or use exit!**: Use 
eturn, 
ext, reak, or raise specific StandardError children. Never shut down SketchUp via exit.

---

## 38. Performance Optimizations (RuboCop Cops)

- **Disable UI during Operations**: Always set disable_ui = true in model.start_operation for heavy changes.
- **Bulk Selection Changes**: model.selection.add(faces) (pass an Array) instead of adding entities one by one in an .each loop.
- **Fast Type Checking**: entity.is_a?(Sketchup::Face) is orders of magnitude faster than entity.typename == 'Face' or entity.class.name == 'Sketchup::Face'.
- **String Encodings**: __FILE__ and __dir__ on Windows might be incorrectly encoded by Ruby. Force UTF-8 if using them for string operations: ile = __FILE__.dup; file.force_encoding('UTF-8').

---

## 39. Creating Entities Safely (RuboCop Cops)

- **Avoid .new for Entities**: Never do Sketchup::Face.new or Sketchup::Group.new. Always use model.active_entities.add_face(...) and dd_group.
- **Group Creation Order**: First create the group group = model.active_entities.add_group, THEN add entities strictly into group.entities. Adding existing geometry into a new group via API is bug-prone and affects performance.
- **Active Entities vs Root**: Prefer model.active_entities over model.entities to ensure your tools obey the user's current context (e.g. inside an open group/component).

---

## 40. Ruby API Bug Avoidance (RuboCop Cops)

- **Material Naming Conflicts**: Setting material.name = 'ExistingName' raises ArgumentError in SU2018+. Always use model.materials.unique_name('Name') before assignment.
- **Render Modes**: Do not use obsolete or invalid rendering modes (e.g. 4 or 99), which crash SketchUp silently.
- **Uniform Scaling Matrices**: Prior to SU2018, Geom::Transformation.scaling(scale) modified the 16th matrix value, breaking some exporters. Workaround/idiom: Prefer Geom::Transformation.scaling(scale, scale, scale).
- **Observer Transparent Operations**: Observers that modify the model (like auto-painting) MUST use transparent operations (start_operation('Name', true, false, true)) so they chain seamlessly into the user's undo stack.


---

## 41. Licensing and Encryption (Extension Warehouse)

To sell your plugin and protect your code, you must combine **Code Encryption (.rbez)** and the **Extension Warehouse Licensing API**.

### 1. Extension Warehouse License API
Always check the license at the moment the command is executed (e.g. inside `perform_action`). Do not hide menus/toolbars if unlicensed (per UX guidelines).

```ruby
def perform_action
  # ⚠️ SECURITY: Store ext_id in a local variable. 
  # Using constants or @instance_vars makes it too easy to tamper with via ObjectSpace.
  ext_id = 'YOUR-EXTENSION-UUID-HERE'
  license = Sketchup::Licensing.get_extension_license(ext_id)
  
  unless license.licensed?
    UI.messagebox('Could not obtain a valid license. Please purchase from Extension Warehouse.')
    return
  end

  # License is valid, proceed...
end
```

### 2. Startup License Check
At the bottom of your `main.rb` file (outside the `file_loaded` block), fetch the license once so SketchUp checks it during startup. Setting this triggers SketchUp's native dialogue if the user's license is missing or expired.

```ruby
# At bottom of main.rb:
# Fetching a license here ensures SketchUp includes the extension in its startup warnings.
ext_id = 'YOUR-EXTENSION-UUID-HERE'
Sketchup::Licensing.get_extension_license(ext_id)
```

### 3. Code Encryption (.rbez) / Ruby Scrambler
Sketchup Ruby files (`.rb`) are plain text. To protect your intellectual property, SketchUp encrypts them into `.rbe` files inside `.rbez` packages.

1. **Development:** Write your code in standard `.rb` files.
2. **Packaging:** Zip your plugin directory and loader file into an `.rbz` file.
3. **Encryption:** Upload the `.rbz` to the **Extension Warehouse Developer Portal**. Choose the option to encrypt the extension before publishing/downloading.
4. **Distribution:** The Extension Warehouse scrambles all your `.rb` files inside the plugin's subfolder into `.rbe`, re-packages them, and serves the `.rbez` to customers. The loader file in the root directory remains plain text.

**Encryption Rules:**
- Never require `.rb` extensions explicitly (e.g., `require 'myplugin/core.rb'`). The `.rb` will be gone after encryption!
- Always omit the extension when requiring files:
  ```ruby
  # ❌ Bad (will fail when encrypted)
  require File.join(PATH, 'core', 'tool.rb')
  
  # ✅ Good (SketchUp automatically finds the .rbe version)
  require File.join(PATH, 'core', 'tool')
  ```
- Make sure your licensing API logic is located inside encrypted secondary files, not the unencrypted loader file in the `Plugins` root folder.


---

## 42. Advanced Custom Tool Lifecycle 

If you are building an interactive Tool that inherits the `Tool` interface, observe these strict practices from the Trimble Official Examples:

### 1. In-Tool Selection (Hover Previews)
Native tools can preview entities visually when hovering without actually adding them to the global selection. The Ruby API *cannot* do this natively. 
**The Workaround:** Track a parallel `@selection` array inside your tool. Only add items to `Sketchup.active_model.selection` when the user actually clicks.
```ruby
# In tool's initialize() or activate():
@selection = Sketchup.active_model.selection.to_a  # Copy to local array

# In onMouseMove():
def onMouseMove(flags, x, y, view)
  if @selection.empty?
    # Try picking the hovered object. This populates view.pick_helper
    # so we know what they are hovering over, but doesn't select it globally yet
    view.model.selection.clear
    pickhelper = view.pick_helper(x, y)
    @hovered_entity = pickhelper.best_picked
  end
  # ...
end
```

### 2. VCB (Measurement Bar) Input Caching
When a user types a value and presses Enter, it triggers `onUserText(text, view)`. However, if they move their mouse *while* typing, `onMouseMove` is continuously called. This causes `update_ui` to overwrite what they are typing with the live length!
**The Workaround:** Cache the VCB value. Only update `vcb_value` if the distance has actually changed.
```ruby
def update_ui
  if picked_first_point?
    Sketchup.status_text = 'Select end point.'
    
    # ⚠️ caching prevents the user's typed text from being erased by mouse jitter
    Sketchup.vcb_value = @distance unless @vcb_cache == @distance
    @vcb_cache = @distance
  else
    Sketchup.status_text = 'Select start point.'
  end
  Sketchup.vcb_label = 'Distance'
end
```

---

## 43. Known InputPoint Bugs in Move Operations

When you move/transform geometry that your `InputPoint` is actively snapping to, the `InputPoint` moves with it, completely destroying your VCB distance calculation and preview lines. This is a known API limitation (Issues #618 and #452).

**The Workaround:** When beginning a drag/move operation, NEVER copy the live `InputPoint`. Instead, instantiate a brand new `InputPoint` pinned to the static 3D coordinate.

```ruby
# ❌ Bad (InputPoint moves with the geometry during drag)
@picked_first_ip.copy!(@mouse_ip)

# ✅ Good (Creates fixed reference point in space)
@picked_first_ip = Sketchup::InputPoint.new(@mouse_ip.position)
```

Additionally, due to a bug in the line stipiling logic when inferencing is locked (Issue #229), thicker lines (e.g. `view.line_width = 3`) cause line dashes (`stipple = "_"`) to scale inconsistently, making your custom tool look un-native.
**The Workaround:** If inference is locked, switch to a solid line (`stipple = ""`) or keep `line_width = 1` if you require dashes.
