# Data-driven design

A designer wants a goblin archer that's the goblin grunt with a bow, less health and more range, and shouldn't have to wait for an engine build to get it. Define entity types (prefabs, blueprints, archetypes) as data, which components, with which parameters, instantiated by a factory, so a designer makes a new enemy variant without an engineer or a recompile[13]. Benefits: iteration speed, designer independence, modding. Unity has prefabs + variants and ScriptableObjects; Unreal has Blueprint classes over a C++ base; Bilas's Dungeon Siege ran over 7,300 object types from data.

Most systems also let one definition inherit from another, so the archer lists only what differs from the grunt. Dungeon Siege's templates did it with a specializes field: each template was compiled on top of its base, overriding the base's fields[13]. Nystrom's prototype example uses the same goblins: the archer names the grunt as its prototype, and any property the archer doesn't have falls back to being looked up on the prototype[14]. Unity's prefab variants follow the same rule: a variant inherits from its base prefab, its overrides take precedence, and a variant can itself be the base of another variant[15].

Worked example

The grunt sets Health.max = 30, Mover.speed = 3.0 and Melee.damage = 6. The archer names the grunt as its prototype and sets only Health.max = 20 and Bow.range = 14. The archer captain names the archer and sets only Health.max = 45. The captain's speed isn't in the captain or the archer, so the lookup reaches the grunt on the third step: 3.0. Patch the grunt's speed to 3.5 and the captain moves at 3.5 without anyone touching the captain's data. Bilas lists the same benefit for placed objects that save only the fields they change: you can "make global changes easily by modifying templates"[13].
trace
grunt speed 3.0
archer overrides speed
captain overrides health
archer_captain resolves to:Health.max 45 (set here) · Mover.speed 3 (from goblin_grunt) · Melee.damage 6 (from goblin_grunt) · Bow.range 14 (from goblin_archer)
Three prefabs, each card showing every field's resolved value and whether the prefab sets it or inherits it (the arrow names the prefab it came from). The highlight walks the lookup for the traced field, starting at the captain. Drag grunt speed and watch both variants follow it; then turn on archer overrides speed and the captain stops at the archer on the second step, and the grunt's slider no longer reaches it. Turn off captain overrides health and the captain inherits the archer's 20. A copy-pasted captain would have kept every old value.

The program below runs the whole path from text to objects. The designer's file names each prefab and its prototype; the parser builds a table; resolveFields walks the chain once per field; and the factory looks each component name up in a registry, the one place where data names meet code.
Prefab variants: parse, resolve through the prototype chain, build components (complete program)
C++
Rust

// Prefabs as data: a variant names its prototype and lists only what it changes.
// A factory resolves each field through the prototype chain and builds components.
#include <cstdio>
#include <functional>
#include <map>
#include <memory>
#include <set>
#include <sstream>
#include <string>
#include <vector>

// What a designer edits. In a game this is a file loaded at startup.
const char* kPrefabText = R"(
[goblin_grunt]
Health.max = 30
Mover.speed = 3.0
Melee.damage = 6

[goblin_archer : goblin_grunt]
Health.max = 20
Bow.range = 14

[archer_captain : goblin_archer]
Health.max = 45
)";

struct Prefab {
    std::string prototype;                      // empty for a root prefab
    std::map<std::string, double> fields;       // "Component.field" -> value
};
using PrefabTable = std::map<std::string, Prefab>;

std::string trim(const std::string& text) {
    std::size_t first = text.find_first_not_of(' '), last = text.find_last_not_of(' ');
    return first == std::string::npos ? "" : text.substr(first, last - first + 1);
}

PrefabTable parsePrefabs(const std::string& text) {
    PrefabTable table;
    Prefab* current = nullptr;
    std::istringstream lines(text);
    for (std::string line; std::getline(lines, line);) {
        if (line.empty()) continue;
        if (line.front() == '[') {                   // [name] or [name : prototype]
            std::string header = line.substr(1, line.find(']') - 1);
            std::size_t colon = header.find(':');
            current = &table[trim(header.substr(0, colon))];
            if (colon != std::string::npos) current->prototype = trim(header.substr(colon + 1));
        } else if (current) {                        // Component.field = value
            std::size_t equals = line.find('=');
            current->fields[trim(line.substr(0, equals))] = std::stod(line.substr(equals + 1));
        }
    }
    return table;
}

// Look in the prefab, then its prototype, then that one's prototype, and so on.
// The first prefab that sets the field wins, so a variant's value overrides its base.
std::map<std::string, double> resolveFields(const PrefabTable& table, const std::string& name) {
    std::map<std::string, double> resolved;
    for (const Prefab* prefab = &table.at(name); prefab;
         prefab = prefab->prototype.empty() ? nullptr : &table.at(prefab->prototype)) {
        for (const auto& [key, value] : prefab->fields)
            resolved.try_emplace(key, value);           // no-op if a nearer prefab set it
    }
    return resolved;
}

struct Component {
    virtual ~Component() = default;
    virtual void describe() const = 0;
};
struct Health : Component {
    double max;
    explicit Health(double maxHealth) : max(maxHealth) {}
    void describe() const override { std::printf("  Health max=%g\n", max); }
};
struct Mover : Component {
    double speed;
    explicit Mover(double moveSpeed) : speed(moveSpeed) {}
    void describe() const override { std::printf("  Mover speed=%g\n", speed); }
};
struct Melee : Component {
    double damage;
    explicit Melee(double hitDamage) : damage(hitDamage) {}
    void describe() const override { std::printf("  Melee damage=%g\n", damage); }
};
struct Bow : Component {
    double range;
    explicit Bow(double bowRange) : range(bowRange) {}
    void describe() const override { std::printf("  Bow range=%g\n", range); }
};

using Fields = std::map<std::string, double>;
using ComponentFactory = std::function<std::unique_ptr<Component>(const Fields&)>;

// The only place that maps data names to code. A new component type registers here once;
// after that, designers can attach it to any prefab without touching C++.
const std::map<std::string, ComponentFactory> kComponentRegistry = {
    {"Health", [](const Fields& fields) { return std::make_unique<Health>(fields.at("Health.max")); }},
    {"Mover",  [](const Fields& fields) { return std::make_unique<Mover>(fields.at("Mover.speed")); }},
    {"Melee",  [](const Fields& fields) { return std::make_unique<Melee>(fields.at("Melee.damage")); }},
    {"Bow",    [](const Fields& fields) { return std::make_unique<Bow>(fields.at("Bow.range")); }},
};

std::vector<std::unique_ptr<Component>> instantiate(const PrefabTable& table, const std::string& name) {
    Fields fields = resolveFields(table, name);
    std::set<std::string> componentNames;                // "Health.max" -> component "Health"
    for (const auto& [key, value] : fields) componentNames.insert(key.substr(0, key.find('.')));
    std::vector<std::unique_ptr<Component>> components;
    for (const std::string& componentName : componentNames)
        components.push_back(kComponentRegistry.at(componentName)(fields));
    return components;
}

int main() {
    PrefabTable table = parsePrefabs(kPrefabText);
    for (const char* name : {"goblin_grunt", "goblin_archer", "archer_captain"}) {
        std::printf("%s\n", name);
        for (const auto& component : instantiate(table, name)) component->describe();
    }
    // Patch the base: every variant that doesn't override the field picks it up.
    table["goblin_grunt"].fields["Mover.speed"] = 3.5;
    std::printf("after the patch, archer_captain speed = %g\n",
                resolveFields(table, "archer_captain").at("Mover.speed"));
}

The output lists each prefab's components: the grunt gets Health, Melee and Mover; the archer and the captain add Bow and keep the grunt's melee and speed; and after the patch the captain's speed reads 3.5. The factory resolves each field once, when it builds the object, so nothing walks the chain during gameplay. Dungeon Siege resolved per template instead of per object: it compiled each template root-down, each one overriding its base's fields, and Bilas recommends keeping the tree flat for performance[13].
What's intentionally missing

    Error reporting. A typo'd prototype or component name, or a value that isn't a number, throws (or panics in Rust) instead of naming the file and line a designer needs to fix.
    Cycle detection. A prefab whose chain leads back to itself makes resolveFields loop forever. A real loader checks the chain when it loads the file.
    A schema. Every field here is a double. Real systems declare each component's fields with types, defaults and docs, which also drives the editor's property sheet (Bilas's system did this).
    Removing inherited parts. A variant here can only add or override. Unity's variants can also remove components and child objects[15].
    Live objects after an edit. Patching the table changes what the factory builds next; objects already in the world keep the values they were built with.

It's a spectrum, with a debuggability cost

Data-driven is not all-or-nothing. Over-data-driving (every constant in config, logic expressed as data graphs) costs debuggability: a crash in a data-defined behavior has no stack frame in your code, no breakpoint, no compile-time type checking until runtime. "No engineer required" is the aspiration; the production reality is a moving line between engine and content. Serializing live game objects (save games) is object-graph serialization with handle remapping on load; the Asset Pipeline tutorial covers the format and versioning machinery.
