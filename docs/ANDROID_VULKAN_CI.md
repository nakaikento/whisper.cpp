# Android Vulkan CI ビルド設定ガイド

GitHub ActionsでwhisperのAndroid Vulkanビルドを行う際に遭遇した問題と解決策をまとめる。

## 概要

whisper.cppのAndroid版でVulkanバックエンドを有効にしてCIビルドする際、複数の依存関係問題が発生した。

## 問題と解決策

### 1. シェーダー生成失敗（glslcがない）

**エラー:**
```
undefined symbol: flash_attn_f32_f16_q4_0_f16acc_len
```

**原因:**
- Vulkanシェーダーは`vulkan-shaders-gen`ツールで生成される
- このツールは内部で`glslc`（GLSLシェーダーコンパイラ）を使用
- GitHub Actions Ubuntuには`glslc`がない

**解決策:**
```yaml
- name: Install Vulkan SDK
  run: |
    wget -qO- https://packages.lunarg.com/lunarg-signing-key-pub.asc | sudo tee /etc/apt/trusted.gpg.d/lunarg.asc
    sudo wget -qO /etc/apt/sources.list.d/lunarg-vulkan-jammy.list https://packages.lunarg.com/vulkan/lunarg-vulkan-jammy.list
    sudo apt update
    sudo apt install -y vulkan-sdk
```

---

### 2. vulkan.hpp not found

**エラー:**
```
fatal error: 'vulkan/vulkan.hpp' file not found
```

**原因:**
- Android NDKにはVulkan Cヘッダー（`vulkan/vulkan.h`）はあるが、C++ヘッダー（`vulkan.hpp`）がない

**解決策:**
Vulkan-Hppをダウンロードしてインクルードパスに追加：
```yaml
- name: Download Vulkan Headers
  run: |
    git clone --depth 1 https://github.com/KhronosGroup/Vulkan-Hpp.git /tmp/Vulkan-Hpp
    mkdir -p external/vulkan
    cp -r /tmp/Vulkan-Hpp/vulkan/* external/vulkan/
```

---

### 3. Vulkanバージョン不一致（1.3 vs 1.4）

**エラー:**
```
error: unknown type name 'VkCopyMemoryToImageInfo'
```

**原因:**
- Android NDK 25.2のVulkanヘッダー: Vulkan 1.3
- 最新Vulkan-Hpp: Vulkan 1.4の型を参照
- 型定義の不一致

**解決策:**
Vulkan-HeadersとVulkan-Hppの両方をダウンロードし、NDKより優先：
```yaml
- name: Download Vulkan Headers
  run: |
    git clone --depth 1 https://github.com/KhronosGroup/Vulkan-Headers.git /tmp/Vulkan-Headers
    git clone --depth 1 https://github.com/KhronosGroup/Vulkan-Hpp.git /tmp/Vulkan-Hpp
    mkdir -p external
    cp -r /tmp/Vulkan-Headers/include/* external/
    cp -r /tmp/Vulkan-Hpp/vulkan/* external/vulkan/

# CMakeLists.txtでBEFORE SYSTEMでNDKより優先
sed -i 's|FetchContent_MakeAvailable(ggml)|include_directories(BEFORE SYSTEM ${CMAKE_SOURCE_DIR}/../../../../../../../external)\nFetchContent_MakeAvailable(ggml)|' CMakeLists.txt
```

---

### 4. vkGetPhysicalDeviceFeatures2 undefined

**エラー:**
```
ld: error: undefined symbol: vkGetPhysicalDeviceFeatures2
>>> did you mean: vkGetPhysicalDeviceFeatures
>>> defined in: .../android/26/libvulkan.so
```

**原因:**
- `vkGetPhysicalDeviceFeatures2`はVulkan 1.1の機能
- Android API 26のlibvulkan.soはVulkan 1.0のみ

**解決策:**
minSdkを29以上に変更（Vulkan 1.1対応）：
```yaml
sed -i 's/minSdk 26/minSdk 29/g' examples/whisper.android/app/build.gradle
sed -i 's/minSdk 26/minSdk 29/g' examples/whisper.android/lib/build.gradle
```

---

### 5. 32bit ARM (armeabi-v7a) でostream問題

**エラー:**
```
error: invalid operands to binary expression ('basic_ostream<char>' and 'vk::Buffer')
```

**原因:**
- Vulkan-Hppの`vk::Buffer`型を`std::cerr`に出力するコードがある
- 32bit ARMではこのoperator<<がオーバーロードされていない

**解決策:**
arm64-v8aのみに限定（Vulkan対応デバイスは基本64bit）：
```yaml
sed -i "s/abiFilters .*/abiFilters 'arm64-v8a'/g" examples/whisper.android/app/build.gradle
sed -i "s/abiFilters .*/abiFilters 'arm64-v8a'/g" examples/whisper.android/lib/build.gradle
```

---

## 最終的なワークフロー構成

```yaml
- name: Install Vulkan SDK
  # glslcシェーダーコンパイラ用

- name: Download Vulkan Headers
  # C + C++ ヘッダー（最新版）

- name: Enable Vulkan in CMakeLists
  # GGML_VULKAN=ON + インクルードパス設定

- name: Update minSdk and ABI
  # minSdk 29 + arm64-v8aのみ
```

## 依存関係まとめ

| コンポーネント | 用途 | 取得元 |
|---------------|------|--------|
| Vulkan SDK | glslc（シェーダーコンパイラ） | LunarG |
| Vulkan-Headers | C headers (vulkan.h等) | Khronos GitHub |
| Vulkan-Hpp | C++ headers (vulkan.hpp等) | Khronos GitHub |

## 注意事項

- **minSdk 29必須**: Vulkan 1.1機能を使用するため
- **arm64-v8aのみ**: 32bitはVulkan-Hppのostream問題あり
- **NDKバージョン**: 25.2.9519653で動作確認済み
