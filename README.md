```text
                                                            oooo
LIB                                                         `888
oooo d8b  .ooooo.   .oooo.o oo.ooooo.   .oooo.    .ooooo.   888  oooo
`888""8P d88' `88b d88(  "8  888' `88b `P  )88b  d88' `"Y8  888 .8P'
 888     888ooo888 `"Y88b.   888   888  .oP"888  888        888888.
 888     888    .o o.  )88b  888   888 d8(  888  888   .o8  888 `88b.
d888b    `Y8bod8P' 8""888P'  888bod8P' `Y888""8o `Y8bod8P' o888o o888o
                             888
                            o888o
```
---

## Main functions
```cpp
// key converters
spk::respack::key_from_string(const std::string& key_str); // std::array<uint8_t, 32>
spk::respack::key_to_string(std::span<const uint8_t> key); // std::string

// packing
spk::respack::pack(const std::filesystem::path& dir, const std::filesystem::path& output_pkg); // spk::respack::res (code, message)
spk::respack::pack(const std::filesystem::path& dir, const std::filesystem::path& output_pkg, std::span<const uint8_t> key); // spk::respack::res (code, message)

// unpacking
spk::respack::unpack(const std::filesystem::path& pkg_path, const std::filesystem::path& output_dir); // spk::respack::res (code, message)
spk::respack::unpack(const std::filesystem::path& pkg_path, const std::filesystem::path& output_dir, std::span<const uint8_t> key); // spk::respack::res (code, message)
// Note: output_dir specifies the directory created to hold unpacked files

// reading from memory (without writing to disk)
spk::respack::read_pack(const std::filesystem::path& pkg_path); // spk::respack::read_res (code, message, map[filename : filecontent])
spk::respack::read_pack(const std::filesystem::path& pkg_path, std::span<const uint8_t> key); // spk::respack::read_res (code, message, map[filename : filecontent])

// reading from memory (lazy)
spk::respack::open_pack(const std::filesystem::path& pkg_path); // spk::respack::open_res (code, message, package)
spk::respack::open_pack(const std::filesystem::path& pkg_path, std::span<const uint8_t> key); // spk::respack::open_res (code, message, package)
```

---

## Usage Example

### Packing

```cpp
#include <iostream>
#include <string>
#include "librespack.hpp"

int main() {
    std::string key_str = "dmPJQedEs4LtX7z55TPzMQk28vrBUrOEkXOrM_8xrrw";
    auto key = spk::respack::key_from_string(key_str);
    auto pack_res = spk::respack::pack("./assets", "assets", key); 
    // first arg is the target dir to pack, second is the package name, third is the key (optional)

    if (!pack_res) {
        std::cerr << "Failed to open pack: " << res.message << "\n";
    }
    std::cout << "Successfully packed resource folder.\n";
    return 0;
}

```

### Unpacking

```cpp
#include <iostream>
#include <string>
#include "librespack.hpp"

int main() {
    std::string key_str = "dmPJQedEs4LtX7z55TPzMQk28vrBUrOEkXOrM_8xrrw"; // random key
    auto key = spk::respack::key_from_string(key_str);
    auto pack_res = spk::respack::unpack("./testfolder.rvlt", "testfolder", key);
    // first arg is the target pack, second is the final directory name, third is the key (optional)
    
    if (!pack_res) {
        std::cerr << "Failed to open pack: " << res.message << "\n";
    }
    std::cout << "Successfully unpacked resource folder.\n";
    return 0;
}

```

### Reading from RAM (Eager)

```cpp
#include <iostream>
#include <string>
#include "librespack.hpp"

namespace fs = std::filesystem;

int main() {
    const std::string archive_path = "assets.rvlt";
    const std::string key_str = "dmPJQedEs4LtX7z55TPzMQk28vrBUrOEkXOrM_8xrrw"; // random key

    // Convert key string to respack key format
    auto key = spk::respack::key_from_string(key_str);

    // Read archive directly into memory map
    auto read_res = spk::respack::read_pack(archive_path, key);
    if (!read_res) {
        std::cerr << "Failed to read pack: " << read_res.message << "\n";
        return 1;
    }

    // Access raw file bytes from RAM
    auto file_it = read_res.files.find("textures/player.png");
    if (file_it != read_res.files.end()) {
        const std::vector<uint8_t>& buffer = file_it->second;
        std::cout << "Loaded file size: " << buffer.size() << " bytes\n";
    }

    // or
    // const auto& filebuffer = read_res.files["textures/player.png"];
    //
    // this works the same way, but the first option is way safer.

    return 0;
}
```

### Reading from RAM (Lazy-load)

```cpp
#include <iostream>
#include <string>
#include "librespack.hpp"

int main() {
    const std::string archive_path = "assets.rvlt";
    const std::string key_str = "dmPJQedEs4LtX7z55TPzMQk28vrBUrOEkXOrM_8xrrw"; // random key

    auto key = spk::respack::key_from_string(key_str);
    auto res = spk::respack::open_pack(archive_path, key);

    if (!res) {
        std::cerr << "Failed to open pack: " << res.message << "\n";
        return 1;
    }

    // Check if the file exists before attempting to read
    if (res.contains("textures/player.png")) {
        // Returns span into the loaded package memory
        auto view = res.get_view("textures/player.png");
        std::cout << "Zero-copy file size: " << view.data.size() << " bytes\n";

        // Allocates and copies bytes into a std::vector<uint8_t> on demand
        auto /*std::vector<uint8_t>*/ buffer = res.get("textures/player.png");
        std::cout << "Copied buffer size: " << buffer.size() << " bytes\n";
    }

    return 0;
}
```

---

## Raylib Integration Example

```cpp
#include <iostream>
#include <string>
#include "librespack.hpp"
#include "raylib.h"

int main() {
    const std::string archive_path = "assets.rvlt";
    const std::string key_str = "dmPJQedEs4LtX7z55TPzMQk28vrBUrOEkXOrM_8xrrw"; // random key

    auto key = spk::respack::key_from_string(key_str);
    auto read_res = spk::respack::read_pack(archive_path, key);

    if (!read_res) {
        std::cerr << "Failed to read pack: " << read_res.message << "\n";
        return 1;
    }

    InitWindow(800, 600, "librespack + Raylib Example");

    Texture2D texture = { 0 };

    auto file_it = read_res.files.find("textures/player.png");
    if (file_it != read_res.files.end()) {
        const std::vector<uint8_t>& buffer = file_it->second; // at the end its always a span of bytes

        // Load image directly from memory buffer
        Image img = LoadImageFromMemory(
            ".png", 
            buffer.data(), 
            static_cast<int>(buffer.size())
        );

        // Upload pixel data to GPU memory
        texture = LoadTextureFromImage(img);

        // Unload CPU-side image data after uploading to GPU
        UnloadImage(img);
    }

    SetTargetFPS(60);

    while (!WindowShouldClose()) {
        BeginDrawing();
        ClearBackground(RAYWHITE);

        if (texture.id > 0) {
            DrawTexture(texture, 100, 100, WHITE);
        }

        EndDrawing();
    }

    UnloadTexture(texture);
    CloseWindow();

    return 0;
}
```

---

### Tips

```cpp
// Custom exit helper using C++23 features
[[noreturn]] void exit(const spk::respack::status& code, const std::string& message) {
    std::cerr << std::format("Error [Code: {}]: {}\n", std::to_underlying(code), message);
    std::exit((int)code);
}

int main() {
    std::string key_str = "dmPJQedEs4LtX7z55TPzMQk28vrBUrOEkXOrM_8xrrw"; // random key
    auto key = spk::respack::key_from_string(key_str);
    auto pack_res = spk::respack::unpack("./testfolder.rvlt", "out", key);
    if (!pack_res) {
        ::exit(pack_res.code, pack_res.message);
    }
    return 0;
}
```

```cpp
// Logging string or text file content
#include <iostream>
#include <string>
#include "librespack.hpp"

// if using read_pack
int main() {
    const std::string archive_path = "assets.rvlt";
    const std::string key_str = "dmPJQedEs4LtX7z55TPzMQk28vrBUrOEkXOrM_8xrrw"; // random key

    auto key = spk::respack::key_from_string(key_str);

    auto read_res = spk::respack::read_pack(archive_path, key);
    if (!read_res) {
        std::cerr << "Failed to read pack: " << read_res.message << "\n";
        return 1;
    }
    auto file_it = read_res.files.find("data/player_data.json");
    if (file_it != read_res.files.end()) {
        const std::vector<uint8_t>& buffer = file_it->second;
        std::string content(buffer.begin(), buffer.end());
        std::cout << content << "\n";
    }

    // or
    // const auto& buffer = read_res.files["data/player_data.json"];
    // std::string content(buffer.begin(), buffer.end());
    // std::cout << content << "\n";
    //
    // this works the same way, but the first option is way safer.

    return 0;
}

// if using open_pack
int main() {
    const std::string archive_path = "assets.rvlt";
    const std::string key_str = "dmPJQedEs4LtX7z55TPzMQk28vrBUrOEkXOrM_8xrrw"; // random key

    auto key = spk::respack::key_from_string(key_str);

    auto res = spk::respack::open_pack(archive_path, key);
    if (!res) {
        std::cerr << "Failed to open pack: " << res.message << "\n";
        return 1;
    }

    const std::string target_file = "data/player_data.json";

    // get_view: Zero-Copy View (Preferred for reading/parsing)
    // Returns: file_view { std::span<uint8_t> data }
    // Allocations: 0
    if (res.contains(target_file)) {
        auto view = res.get_view(target_file);
        if (view) {
            std::string_view content(view.data.begin(), view.data.end());
            std::cout << "[get_view]\n" << content << "\n\n";
        }
    }

    // get: Direct Shorthand
    // Returns: std::vector<uint8_t> (empty vector if failed)
    // Allocations: Copies data into a new vector
    if (res.contains(target_file)) {
        auto buffer = res.get(target_file);
        if (!buffer.empty()) {
            std::string content(buffer.begin(), buffer.end());
            std::cout << "[get]\n" << content << "\n\n";
        }
    }

    // read_file(filename): Explicit Result with Error Details
    // Returns: file_res { code, message, data }
    // Allocations: Copies data into a new vector inside file_res
    if (res.contains(target_file)) { // u can avoid this statement
        auto file_res = res.read_file(target_file);
        if (file_res) {
            std::string content(file_res.data.begin(), file_res.data.end());
            std::cout << "[read_file]\n" << content << "\n\n";
        } else {
            std::cerr << "Error reading " << target_file << ": " << file_res.message << "\n";
        }
    }

    // read_file(filename, out_data): Buffer Reuse (Allocation Efficient)
    // Returns: res { code, message }
    // Allocations: Reuses an existing std::vector<uint8_t>
    if (res.contains(target_file)) { // u can avoid this statement
        std::vector<uint8_t> buffer;
        auto r = res.read_file(target_file, buffer);
        if (r) {
            std::string content(buffer.begin(), buffer.end());
            std::cout << "[read_file with out parameter]\n" << content << "\n\n";
        } else {
            std::cerr << "Error reading " << target_file << ": " << r.message << "\n";
        }
    }

    return 0;
}
```
> TODO: Create C lib.