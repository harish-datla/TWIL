# Motion matching

Hand-authoring a state and a tuned transition for every start, stop, turn and foot plant gets expensive as the move set grows. Motion matching (Büttner and Clavet at Ubisoft, shipped in For Honor) is a data-driven alternative to a hand-built state machine[11][12]. Instead of authoring states and transitions, you keep a database of mocap frames, each tagged with a feature vector (the future trajectory, plus foot positions/velocities and hip velocity). Each search builds a query vector from the desired trajectory plus the current pose, finds the nearest database frame by a weighted cost, and blends to it (For Honor cross-faded; later systems often use inertialization).

Holden and colleagues' paper on Learned Motion Matching spells out a concrete version, the one the authors say is used in several AAA game productions[12]. Every frame of the database gets a feature vector of 27 numbers, all measured relative to the character: where its root will be on the ground 20, 40 and 60 frames ahead at 60 Hz (6 numbers), which way it will face then (6), its two feet's positions (6) and velocities (6), and its hip velocity (3). Because metres, unit directions and velocities have very different ranges, each feature is divided by its standard deviation over the database before distances are taken. Every few frames, or when the stick moves sharply, the game builds a query in the same format (the trajectory part from a spring-damper prediction of the stick, the pose part from the frame playing now) and searches for the frame with the smallest squared distance. If that beats the frame already playing, playback jumps there and an inertialization blend hides the jump[12].

Worked example

Twenty minutes of mocap at 60 frames a second is 72,000 frames. With 27 numbers per frame, a brute-force search compares the query against 72,000 × 27 ≈ 1.9 million numbers. That's why searches run every few frames rather than every frame, and why production systems add an acceleration structure. The Learned Motion Matching paper names a KD-tree (citing Clavet's talk) among the options and itself uses bounding boxes around runs of 16 and 64 consecutive frames, skipping any box whose nearest point is already farther than the best match so far[12].

The widget and the listing below shrink this to root motion only: how the character travels across the ground, with no skeleton. Nine synthetic clips (idle, walk, run, turns at two speeds, a start and a stop) are generated as ground trajectories at 60 Hz. Each frame's feature vector has 13 numbers: the same 6 numbers of future position and 6 of future facing as above, plus the current speed. That leaves 1,380 frames that can be matched, since the last second of each clip has no full second of future to describe.
auto stick
speed
turn
searches/s 4
playing: run_turn_left frame 26wanted: 4 m/s, -45°/seach search: 1380 × 13 numbersbest cost 0.00searches 945, clip changes 80
Top-down: the character walks the grid, the amber rings are the query (where the wanted motion would take it 1/3, 2/3 and 1 s from now), and the green dots are where the playing frame's own motion goes. On a wide screen the side panel shows which clip and frame are playing. Let the auto stick run, then drop the search rate to 1 a second and set the stick by hand. When the green dots sit on the amber rings, the playing frame already does what's wanted. After the stick changes, the character keeps playing the old clip until the next search, so at 1 search a second it can keep running straight for up to a second after you ask for a turn. Ask for 3 m/s turning right at 30°/s and no clip matches exactly: the search settles on the run turning right, the closest frame by the scaled distance. This widget searches only on its schedule; a production system also searches when the stick changes sharply.
Motion matching search: features, per-group scaling, brute force with early out (complete program)
C++
Rust

// Motion matching, the search step: describe every database frame with a
// feature vector, scale each feature group by its spread so none dominates,
// then find the frame closest to a query. Root motion only, no skeleton.
#include <array>
#include <cmath>
#include <cstdio>
#include <limits>
#include <numbers>
#include <string>
#include <vector>

constexpr int kFps = 60;
constexpr int kFutureFrames[3] = {20, 40, 60};   // 1/3, 2/3 and 1 s ahead
constexpr int kFeatureCount = 13;               // 6 trajectory positions, 6 facings, 1 speed
using Features = std::array<float, kFeatureCount>;
constexpr float kRadiansPerDegree = std::numbers::pi_v<float> / 180.0f;

// A clip's root motion, one sample per frame. Facing is the direction
// (sin yaw, cos yaw) on the ground; a positive turn rate turns right.
struct Clip { std::string name; std::vector<float> x, z, yaw, speed; };

template <class SpeedFn, class TurnFn>
Clip makeClip(std::string name, float seconds, SpeedFn speedAt, TurnFn turnDegreesAt) {
    Clip clip{std::move(name), {}, {}, {}, {}};
    float x = 0, z = 0, yaw = 0;
    for (int frame = 0; frame < static_cast<int>(seconds * kFps); ++frame) {
        float time = static_cast<float>(frame) / kFps, speed = speedAt(time);
        clip.x.push_back(x); clip.z.push_back(z); clip.yaw.push_back(yaw); clip.speed.push_back(speed);
        yaw += turnDegreesAt(time) * kRadiansPerDegree / kFps;
        x += std::sin(yaw) * speed / kFps;
        z += std::cos(yaw) * speed / kFps;
    }
    return clip;
}

// Everything is measured in the character's own frame (x right, z forward),
// so a left turn looks the same wherever the character stands.
Features featuresAt(const Clip& clip, int frame) {
    Features features{};
    float cosYaw = std::cos(clip.yaw[frame]), sinYaw = std::sin(clip.yaw[frame]);
    for (int k = 0; k < 3; ++k) {
        int future = frame + kFutureFrames[k];
        float dx = clip.x[future] - clip.x[frame], dz = clip.z[future] - clip.z[frame];
        features[2 * k]     = dx * cosYaw - dz * sinYaw;           // to the right
        features[2 * k + 1] = dx * sinYaw + dz * cosYaw;           // ahead
        float turned = clip.yaw[future] - clip.yaw[frame];
        features[6 + 2 * k]     = std::sin(turned);                // future facing, local
        features[6 + 2 * k + 1] = std::cos(turned);
    }
    features[12] = clip.speed[frame];
    return features;
}

constexpr int groupOf(int feature) { return feature < 6 ? 0 : feature < 12 ? 1 : 2; }

struct Database {
    std::vector<Features> rows;
    std::vector<int> clipOf, frameOf;
    std::array<float, 3> groupScale{};          // 1 / standard deviation of each feature group

    explicit Database(const std::vector<Clip>& clips) {
        for (int clipIndex = 0; clipIndex < static_cast<int>(clips.size()); ++clipIndex)
            // Only frames with a full second of future can be described, so the last 60 can't be matched.
            for (int frame = 0; frame + kFutureFrames[2] < static_cast<int>(clips[clipIndex].x.size()); ++frame) {
                rows.push_back(featuresAt(clips[clipIndex], frame));
                clipOf.push_back(clipIndex); frameOf.push_back(frame);
            }
        // Metres, unit vectors and m/s have different spreads; without scaling, the
        // positions (up to 4 m) would swamp the facings (at most 2 apart).
        std::array<double, kFeatureCount> mean{};
        for (const Features& row : rows) for (int d = 0; d < kFeatureCount; ++d) mean[d] += row[d] / static_cast<double>(rows.size());
        std::array<double, 3> variance{}, count{};
        for (const Features& row : rows)
            for (int d = 0; d < kFeatureCount; ++d) {
                variance[groupOf(d)] += (row[d] - mean[d]) * (row[d] - mean[d]);
                count[groupOf(d)] += 1;
            }
        for (int group = 0; group < 3; ++group)
            groupScale[group] = static_cast<float>(1.0 / std::sqrt(variance[group] / count[group]));
    }

    // Brute force: one scaled squared distance per frame.
    int search(const Features& query, float& bestCost) const {
        int best = -1;
        bestCost = std::numeric_limits<float>::max();
        for (int row = 0; row < static_cast<int>(rows.size()); ++row) {
            float cost = 0;
            for (int d = 0; d < kFeatureCount; ++d) {
                float difference = (query[d] - rows[row][d]) * groupScale[groupOf(d)];
                cost += difference * difference;
                if (cost >= bestCost) break;                         // can't win: stop early
            }
            if (cost < bestCost) { bestCost = cost; best = row; }
        }
        return best;
    }
};

// A query from desired motion: where the character would be over the next second
// if it held this speed and turn rate. Games predict this from the stick with a spring.
Features queryFor(float speed, float turnDegreesPerSecond) {
    Clip prediction = makeClip("query", 1.1f, [=](float) { return speed; },
                               [=](float) { return turnDegreesPerSecond; });
    return featuresAt(prediction, 0);
}

int main() {
    auto constant = [](float value) { return [=](float) { return value; }; };
    std::vector<Clip> clips = {
        makeClip("idle", 2, constant(0.0f), constant(0.0f)),
        makeClip("walk", 4, constant(1.4f), constant(0.0f)),
        makeClip("run", 4, constant(4.0f), constant(0.0f)),
        makeClip("walk_turn_right", 4, constant(1.4f), constant(60.0f)),
        makeClip("walk_turn_left", 4, constant(1.4f), constant(-60.0f)),
        makeClip("run_turn_right", 4, constant(4.0f), constant(45.0f)),
        makeClip("run_turn_left", 4, constant(4.0f), constant(-45.0f)),
        makeClip("start", 3, [](float seconds) { return 1.4f * std::fmin(1.0f, seconds); }, constant(0.0f)),
        makeClip("stop", 3, [](float seconds) { return 1.4f * std::fmin(1.0f, std::fmax(0.0f, 2.0f - seconds)); }, constant(0.0f)),
    };
    Database database(clips);
    std::printf("%zu matchable frames x %d features = %zu multiply-adds per full search\n",
                database.rows.size(), kFeatureCount, database.rows.size() * kFeatureCount);

    float cost = 0;
    int known = 700;                                               // a query copied from the database finds itself
    int found = database.search(database.rows[known], cost);
    std::printf("row %d -> row %d, cost %.3f\n", known, found, cost);

    const float wishes[][2] = {{1.4f, 0}, {4.0f, 0}, {1.4f, -60}, {3.0f, 30}, {0.0f, 0}};
    for (const auto& wish : wishes) {
        int row = database.search(queryFor(wish[0], wish[1]), cost);
        std::printf("want %.1f m/s turning %+4.0f deg/s -> %-15s frame %3d, cost %.3f\n", wish[0], wish[1],
                    clips[database.clipOf[row]].name.c_str(), database.frameOf[row], cost);
    }
}

The output shows the parts working: a query copied from row 700 finds row 700 at cost 0; wishes that match a clip exactly find it at cost 0 (at frame 0, since every frame of a constant clip looks the same); and 3 m/s turning right at 30°/s, which no clip covers, lands in the run turning right at cost 4.2. The early out stops summing a row as soon as its partial cost passes the best so far, the same trick the bounding-box search relies on.
What's intentionally missing

The listing keeps only the search. A production matcher also has:

    pose features (foot positions and velocities, hip velocity) read from the skeleton, so a jump lands on a frame whose legs match the ones on screen;
    a query trajectory predicted from the stick by a spring, instead of a fixed speed and turn rate;
    per-group weights on top of the scaling, tuned by hand;
    an acceleration structure (bounding boxes or a KD-tree) instead of a full scan;
    tags or database ranges that limit the search to the gait or action the game asks for;
    an inertialization blend on every jump, and a search whenever the stick changes sharply.

Not free, and it didn't kill the state machine

The search often runs only a few times a second rather than every frame (O3DE's implementation, for one[13]); between searches the chosen clip simply plays, and a blend smooths each jump. It's not authoring-free: someone has to capture and curate a large, clean mocap database, shot from a scripted "dance card" of walks, plants, and strafes at a spread of angles. The nearest-neighbor search has real cost, usually cut with KD-trees or similar structures[13]. And it does not replace state machines everywhere: in For Honor the game logic still ran as a simple timed state machine, which requested motion from the matcher[11]. Feature-vector contents vary by engine; there is no single canonical schema.
