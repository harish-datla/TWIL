# Composition: three models

Once objects are built from parts, the next decision is what a part is and who calls it. The fix: the entity becomes a container of components, each owning one domain (transform, render, physics, AI)[2]. But composition splits into three different models, each shipping AAA games, and they should not be collapsed into one:
ModelShapeTradeoff
Unity: GameObject + MonoBehaviourAddComponent<T>(); components are heap C# objects[3]Easy, flexible; polymorphic + pointer-chased (not data-oriented)
Unreal: Actor + ActorComponentComposition plus a shallow inheritance spine[4]Purposeful inheritance + components; ships many of the largest AAA games
Pure ECSEntity = ID, component = data, system = functionCache-friendly storage; see the ECS tutorial
Don't crown ECS

ECS is one of three live models, not "the modern/correct" one. Unreal is composition with inheritance; Unity is GameObject+component (with DOTS/ECS optional for hot paths). And Unity-style "everything is a component" composition is not the same as data-oriented ECS: those components are still polymorphic, heap-allocated, and pointer-chased; the ECS cache win comes from storage layout, not from "using components." Pick the model that fits the game.

Worked example

Ten thousand objects with four components each, in the Unity-style listing below, cost 40,000 virtual update calls a frame, each on a separately allocated component. The same behavior in an ECS is four system loops, each walking one packed array. Both are composition; the difference is where the data lives and who loops over it.
A component-based game object
C++
Rust

struct Component { virtual void update(float dt) = 0; virtual ~Component() = default; };

class GameObject {
    std::vector<std::unique_ptr<Component>> components;
public:
    template<class T, class... A> T* add(A&&... a) {       // composition: attach behavior
        auto c = std::make_unique<T>(std::forward<A>(a)...);
        T* raw = c.get(); components.push_back(std::move(c)); return raw;
    }
    void update(float dt) { for (auto& c : components) c->update(dt); }
};   // Unity-style: clear, but polymorphic + heap-allocated (not the ECS layout)

