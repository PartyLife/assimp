# ASE File Parsing Workflow in Assimp

## Overview

The ASE (ASCII Scene Export) importer in Assimp is responsible for reading and parsing 3D model files exported from 3ds Max in ASCII format. This document explains the complete workflow of how the ASE plugin processes `.ase` and `.ask` files.

## Architecture

The ASE importer consists of two main components:

1. **ASEParser** (`code/AssetLib/ASE/ASEParser.h/cpp`) - Responsible for parsing the ASCII file format
2. **ASEImporter** (`code/AssetLib/ASE/ASELoader.h/cpp`) - Responsible for converting parsed data into Assimp's internal scene structure

## File Format

ASE files are text-based 3D scene descriptions exported from 3ds Max. The format uses a hierarchical structure with blocks denoted by curly braces `{}` and keywords prefixed with asterisks `*`.

### Example ASE File Structure:
```
*3DSMAX_ASCIIEXPORT 200
*SCENE {
    *SCENE_FILENAME ""
    *SCENE_FIRSTFRAME 0
    *SCENE_LASTFRAME 100
    *SCENE_FRAMESPEED 30
}
*MATERIAL_LIST {
    *MATERIAL_COUNT 1
    *MATERIAL 0 {
        *MATERIAL_NAME "Material01"
        *MATERIAL_DIFFUSE 1.0 0.0 0.0
    }
}
*GEOMOBJECT {
    *NODE_NAME "Box01"
    *MESH {
        *MESH_NUMVERTEX 8
        *MESH_NUMFACES 12
        *MESH_VERTEX_LIST { ... }
        *MESH_FACE_LIST { ... }
    }
}
```

## Data Structures

### Key ASE Data Structures

1. **BaseNode**: Base class for all scene nodes (meshes, lights, cameras, dummies)
2. **Mesh**: Represents a 3D mesh with vertices, faces, normals, UVs, and vertex colors
3. **Material**: Material properties including colors, textures, and shading properties
4. **Light**: Light source with type, color, intensity, and animation data
5. **Camera**: Camera with FOV, near/far planes, and animation data
6. **Animation**: Keyframe animation data for position, rotation, and scaling

## Visual Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ASE File Import Workflow                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────┐
│  ASE File (.ase)│
│  Text-based 3D  │
│  Scene Export   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: File Loading (ASEImporter::InternReadFile)                  │
│                                                                       │
│  • Open file via IOSystem                                            │
│  • Read entire file into memory buffer                               │
│  • Determine format version (.asc=v110, .ase=v200)                  │
│  • Create ASE::Parser instance with buffer                           │
└────────┬──────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Token-Based Parsing (Parser::Parse)                         │
│                                                                       │
│  Main parsing loop processes hierarchical blocks:                    │
│  ┌────────────────────────────────────────────────────┐             │
│  │ *3DSMAX_ASCIIEXPORT → File format version         │             │
│  │ *SCENE              → Scene settings               │             │
│  │ *MATERIAL_LIST      → All materials                │             │
│  │ *GEOMOBJECT         → Mesh objects                 │             │
│  │ *LIGHTOBJECT        → Light sources                │             │
│  │ *CAMERAOBJECT       → Camera objects               │             │
│  │ *HELPEROBJECT       → Dummy/helper objects         │             │
│  └────────────────────────────────────────────────────┘             │
│                                                                       │
│  Parsing Hierarchy:                                                  │
│  Level 1 (LV1) → Top-level blocks (scene, objects, materials)       │
│  Level 2 (LV2) → Object properties (transform, mesh, animation)     │
│  Level 3 (LV3) → Detailed data (vertices, faces, UVs, normals)      │
│  Level 4 (LV4) → Primitive values (floats, integers, vectors)       │
│                                                                       │
│  Output: Populated data structures in Parser members:               │
│  • m_vMeshes (vector<Mesh>)                                          │
│  • m_vMaterials (vector<Material>)                                   │
│  • m_vLights (vector<Light>)                                         │
│  • m_vCameras (vector<Camera>)                                       │
│  • m_vDummies (vector<Dummy>)                                        │
└────────┬──────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Mesh Processing                                             │
│                                                                       │
│  For each mesh in m_vMeshes:                                         │
│  ┌─────────────────────────────────────────────────────┐            │
│  │ BuildUniqueRepresentation()                         │            │
│  │  • Eliminate duplicate vertices                     │            │
│  │  • Create unified vertex/normal/UV/color indices    │            │
│  │  • Ensure each vertex is unique                     │            │
│  └─────────────────────────────────────────────────────┘            │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────────────────┐            │
│  │ GenerateNormals()                                   │            │
│  │  • Check if normals exist in file                   │            │
│  │  • Generate smooth normals based on smoothing groups│            │
│  │  • Optional: Force recompute via config flag        │            │
│  └─────────────────────────────────────────────────────┘            │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────────────────┐            │
│  │ ConvertMeshes()                                     │            │
│  │  • Split mesh by material (one aiMesh per material) │            │
│  │  • Create vertex buffers (pos, normal, UV, color)   │            │
│  │  • Set up face indices                              │            │
│  │  • Assign material indices                          │            │
│  └─────────────────────────────────────────────────────┘            │
│                                                                       │
│  Output: vector<aiMesh*> avOutMeshes                                │
└────────┬──────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Material Processing                                         │
│                                                                       │
│  BuildMaterialIndices()                                              │
│  • Resolve material references                                       │
│  • Handle submaterials                                               │
│  • Create final material list                                        │
│  • Convert ASE::Material → aiMaterial                                │
│                                                                       │
│  For each material:                                                  │
│  • Set colors (diffuse, specular, ambient)                           │
│  • Set material properties (shininess, transparency)                 │
│  • Load texture maps (diffuse, specular, bump, etc.)                 │
│                                                                       │
│  Output: scene->mMaterials[], scene->mNumMaterials                   │
└────────┬──────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Scene Graph Construction                                    │
│                                                                       │
│  Collect all nodes:                                                  │
│  nodes = [meshes] + [lights] + [cameras] + [dummies]                │
│                                                                       │
│  BuildNodes()                                                        │
│  • Create hierarchical node structure (aiNode tree)                  │
│  • Process parent-child relationships                                │
│  • Apply transformation matrices                                     │
│  • Attach meshes to nodes                                            │
│                                                                       │
│  Node Hierarchy Example:                                             │
│  RootNode                                                            │
│    ├─ Node1 (Mesh: Box01)                                            │
│    ├─ Node2 (Mesh: Sphere01)                                         │
│    │   └─ Node3 (Light: Light01)                                     │
│    └─ Node4 (Camera: Camera01)                                       │
│                                                                       │
│  Output: scene->mRootNode (complete node hierarchy)                  │
└────────┬──────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6: Animation Processing                                        │
│                                                                       │
│  BuildAnimations()                                                   │
│  • Create aiAnimation objects                                        │
│  • Set up animation channels for each animated node                  │
│  • Convert keyframe data:                                            │
│    - Position keys (akeyPositions)                                   │
│    - Rotation keys (akeyRotations)                                   │
│    - Scaling keys (akeyScaling)                                      │
│  • Handle animation types (Track, Bezier, TCB)                       │
│  • Process target animations for lights/cameras                      │
│                                                                       │
│  Output: scene->mAnimations[], scene->mNumAnimations                 │
└────────┬──────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 7: Asset Creation (Lights & Cameras)                           │
│                                                                       │
│  BuildLights()                                                       │
│  • Convert ASE::Light → aiLight                                      │
│  • Set light type (omni, directional, spot)                          │
│  • Set color and intensity                                           │
│  • Set position and direction                                        │
│  • Set attenuation parameters                                        │
│                                                                       │
│  BuildCameras()                                                      │
│  • Convert ASE::Camera → aiCamera                                    │
│  • Set FOV (field of view)                                           │
│  • Set near/far clip planes                                          │
│  • Set position and look-at direction                                │
│  • Handle target cameras                                             │
│                                                                       │
│  Output:                                                             │
│  • scene->mLights[], scene->mNumLights                               │
│  • scene->mCameras[], scene->mNumCameras                             │
└────────┬──────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ FINAL OUTPUT: Complete aiScene Structure                            │
│                                                                       │
│  aiScene                                                             │
│  ├─ mRootNode       → Scene hierarchy (tree of aiNode)               │
│  ├─ mMeshes[]       → Array of aiMesh objects                        │
│  ├─ mMaterials[]    → Array of aiMaterial objects                    │
│  ├─ mAnimations[]   → Array of aiAnimation objects                   │
│  ├─ mLights[]       → Array of aiLight objects                       │
│  ├─ mCameras[]      → Array of aiCamera objects                      │
│  ├─ mTextures[]     → Embedded textures (if any)                     │
│  └─ mMetaData       → Scene metadata                                 │
│                                                                       │
│  Ready for use in your application!                                  │
└─────────────────────────────────────────────────────────────────────┘
```

## Complete Workflow

### 1. File Loading (ASEImporter::InternReadFile)

```
Input: File path and IOSystem
↓
Open and read file into memory buffer
↓
Determine file format version (110 for .asc, 200 for .ase)
↓
Create ASE::Parser instance
```

**Location**: `code/AssetLib/ASE/ASELoader.cpp:115`

**Key steps**:
- Opens the `.ase` file using IOSystem
- Reads entire file content into memory buffer
- Determines format version based on file extension (ASC = v110, ASE = v200)
- Creates parser instance with buffer and format version

### 2. Parsing Phase (Parser::Parse)

```
Parser initialization
↓
Main parsing loop - process tokens
├── *3DSMAX_ASCIIEXPORT → Read format version
├── *SCENE → ParseLV1SceneBlock()
├── *MATERIAL_LIST → ParseLV1MaterialListBlock()
├── *GEOMOBJECT → ParseLV1ObjectBlock() for meshes
├── *LIGHTOBJECT → ParseLV1ObjectBlock() for lights
├── *CAMERAOBJECT → ParseLV1ObjectBlock() for cameras
└── *HELPEROBJECT → ParseLV1ObjectBlock() for dummies
↓
Parsed data stored in Parser's member vectors
```

**Location**: `code/AssetLib/ASE/ASEParser.cpp:231`

**Parsing hierarchy**:

- **Level 1 (LV1)**: Top-level blocks
  - `ParseLV1SceneBlock()`: Scene settings (frame range, speed, background)
  - `ParseLV1MaterialListBlock()`: All materials in the scene
  - `ParseLV1ObjectBlock()`: Individual objects (meshes, lights, cameras, dummies)

- **Level 2 (LV2)**: Object-level blocks
  - `ParseLV2MaterialBlock()`: Individual material properties
  - `ParseLV2NodeTransformBlock()`: Node transformation matrix
  - `ParseLV2AnimationBlock()`: Animation data
  - `ParseLV2MeshBlock()`: Mesh geometry data
  - `ParseLV2LightSettingsBlock()`: Light settings
  - `ParseLV2CameraSettingsBlock()`: Camera settings

- **Level 3 (LV3)**: Detailed data blocks
  - `ParseLV3MeshVertexListBlock()`: Vertex positions
  - `ParseLV3MeshFaceListBlock()`: Face indices
  - `ParseLV3MeshTListBlock()`: Texture coordinates
  - `ParseLV3MeshTFaceListBlock()`: Texture face indices
  - `ParseLV3MeshCListBlock()`: Vertex colors
  - `ParseLV3MeshNormalListBlock()`: Vertex normals
  - `ParseLV3MapBlock()`: Texture maps
  - Animation blocks: Position, rotation, scaling keyframes

- **Level 4 (LV4)**: Primitive data parsing
  - `ParseLV4MeshRealTriple()`: Parse 3 float values (positions, normals)
  - `ParseLV4MeshLongTriple()`: Parse 3 integer values (face indices)
  - `ParseLV4MeshFace()`: Parse individual face data

### 3. Data Conversion (ASEImporter::InternReadFile continued)

After parsing completes, the data is converted to Assimp's internal format:

```
Parsed ASE data
↓
For each mesh:
├── BuildUniqueRepresentation() - Create unique vertices
├── GenerateNormals() - Generate/validate normals
└── ConvertMeshes() - Convert to aiMesh objects
↓
BuildMaterialIndices() - Setup material references
↓
Collect all nodes (meshes, lights, cameras, dummies)
↓
BuildNodes() - Create scene hierarchy
↓
BuildAnimations() - Setup animation channels
↓
BuildCameras() - Create camera objects
↓
BuildLights() - Create light objects
↓
Complete aiScene structure
```

**Location**: `code/AssetLib/ASE/ASELoader.cpp:150-245`

#### 3.1 Mesh Processing

**BuildUniqueRepresentation()**: 
- Eliminates duplicate vertices
- Creates unified vertex/normal/UV/color indices
- Ensures each vertex is unique in the final mesh

**GenerateNormals()**:
- Checks if normals exist in the file
- Generates smooth normals based on smoothing groups if needed
- Can be configured to always recompute normals via `AI_CONFIG_IMPORT_ASE_RECONSTRUCT_NORMALS`

**ConvertMeshes()**:
- Splits mesh by material (one aiMesh per material)
- Creates vertex buffers with positions, normals, UVs, colors
- Sets up face indices
- Assigns material indices

#### 3.2 Scene Graph Construction

**BuildNodes()**:
- Creates hierarchical node structure
- Processes parent-child relationships
- Applies transformation matrices
- Attaches meshes to nodes

**BuildAnimations()**:
- Creates aiAnimation objects
- Sets up animation channels for each animated node
- Converts keyframe data (position, rotation, scaling)
- Handles target animations for lights and cameras

#### 3.3 Asset Creation

**BuildMaterialIndices()**:
- Resolves material references
- Handles submaterials
- Creates final material list

**BuildCameras()**:
- Converts ASE cameras to aiCamera objects
- Sets FOV, near/far planes
- Handles target cameras

**BuildLights()**:
- Converts ASE lights to aiLight objects
- Sets light type (omni, directional, spot)
- Applies color and intensity

### 4. Output

The final output is a complete `aiScene` structure containing:
- Scene hierarchy (nodes)
- Meshes with geometry data
- Materials with textures
- Lights
- Cameras
- Animations

## Key Features

### Supported ASE Elements

✓ **Geometry**
- Vertex positions
- Face definitions
- Normals (with smoothing groups)
- Multiple UV channels (up to `AI_MAX_NUMBER_OF_TEXTURECOORDS`)
- Vertex colors
- Material indices per face

✓ **Materials**
- Diffuse, specular, ambient colors
- Transparency
- Shininess
- Texture maps (diffuse, specular, bump, etc.)
- Submaterials (multi-material support)

✓ **Scene Elements**
- Lights (omni, target, directional, free)
- Cameras (free, target)
- Dummy objects (helpers)
- Scene hierarchy with transformations

✓ **Animation**
- Position keyframes
- Rotation keyframes
- Scaling keyframes
- Multiple animation types (Track, Bezier, TCB)
- Target animations for lights/cameras

✓ **Skinning** (partial support)
- Bone definitions
- Vertex weights

### File Format Versions

- **Version 110**: Older format (`.asc` extension)
- **Version 200**: Current format (`.ase` extension)

The parser automatically detects and handles both versions.

## Configuration Options

- `AI_CONFIG_IMPORT_ASE_RECONSTRUCT_NORMALS` (default: 1)
  - When enabled, always recomputes normals regardless of file content
  - Useful workaround for broken ASE normal exports from 3DS Max

- `AI_CONFIG_IMPORT_NO_SKELETON_MESHES` (default: 0)
  - Controls whether to generate skeleton visualization meshes

## Error Handling

The parser includes robust error handling:
- Validates file structure and token sequences
- Checks for unexpected EOF
- Warns about inconsistencies (vertex/face counts)
- Validates matching braces
- Line number tracking for error reporting

## Implementation Details

### Token Parsing

The parser uses a simple token-based approach:
- `SkipToNextToken()`: Advances to next `*`, `{`, or `}` character
- `SkipSection()`: Skips entire blocks including nested subsections
- `TokenMatch()`: Checks if current position matches a keyword
- Macros `AI_ASE_HANDLE_TOP_LEVEL_SECTION()` and `AI_ASE_HANDLE_SECTION()` manage block depth

### Memory Management

- File is loaded entirely into memory for fast random access
- Parser maintains pointer to current position in buffer
- No dynamic allocation during parsing (uses pre-allocated vectors)
- Efficient string handling using C-style pointers

### Optimization

- Single-pass parsing
- Minimal string copying
- Direct memory buffer access
- Pre-allocated data structures

## Testing

Test files are located in:
- `test/models/ASE/` - Valid test files
- `test/models-nonbsd/ASE/` - Additional test files
- `test/models/invalid/` - Invalid test cases

Test coverage includes:
- Basic geometry import
- Animation import
- Camera animation
- Multiple materials
- UTF-16 encoded files
- Edge cases and error conditions

---

# Quy Trình Làm Việc Của Plugin ASE Trong Assimp (Vietnamese)

## Tổng Quan

Plugin ASE (ASCII Scene Export) trong Assimp có trách nhiệm đọc và phân tích các file mô hình 3D được xuất từ 3ds Max ở định dạng ASCII. Tài liệu này giải thích quy trình hoàn chỉnh về cách plugin ASE xử lý các file có đuôi `.ase` và `.ask`.

## Kiến Trúc

Plugin ASE bao gồm hai thành phần chính:

1. **ASEParser** (`code/AssetLib/ASE/ASEParser.h/cpp`) - Chịu trách nhiệm phân tích định dạng file ASCII
2. **ASEImporter** (`code/AssetLib/ASE/ASELoader.h/cpp`) - Chịu trách nhiệm chuyển đổi dữ liệu đã phân tích thành cấu trúc scene nội bộ của Assimp

## Định Dạng File

File ASE là các file văn bản mô tả scene 3D được xuất từ 3ds Max. Định dạng sử dụng cấu trúc phân cấp với các khối được đánh dấu bằng dấu ngoặc nhọn `{}` và các từ khóa có tiền tố là dấu sao `*`.

### Ví Dụ Cấu Trúc File ASE:
```
*3DSMAX_ASCIIEXPORT 200
*SCENE {
    *SCENE_FILENAME ""
    *SCENE_FIRSTFRAME 0
    *SCENE_LASTFRAME 100
    *SCENE_FRAMESPEED 30
}
*MATERIAL_LIST {
    *MATERIAL_COUNT 1
    *MATERIAL 0 {
        *MATERIAL_NAME "Material01"
        *MATERIAL_DIFFUSE 1.0 0.0 0.0
    }
}
*GEOMOBJECT {
    *NODE_NAME "Box01"
    *MESH {
        *MESH_NUMVERTEX 8
        *MESH_NUMFACES 12
        *MESH_VERTEX_LIST { ... }
        *MESH_FACE_LIST { ... }
    }
}
```

## Cấu Trúc Dữ Liệu

### Các Cấu Trúc Dữ Liệu ASE Chính

1. **BaseNode**: Lớp cơ sở cho tất cả các node trong scene (meshes, lights, cameras, dummies)
2. **Mesh**: Đại diện cho một lưới 3D với vertices, faces, normals, UVs, và màu vertex
3. **Material**: Thuộc tính vật liệu bao gồm màu sắc, texture, và thuộc tính shading
4. **Light**: Nguồn sáng với loại, màu, cường độ, và dữ liệu animation
5. **Camera**: Camera với FOV, near/far planes, và dữ liệu animation
6. **Animation**: Dữ liệu animation keyframe cho vị trí, xoay, và tỉ lệ

## Quy Trình Hoàn Chỉnh

### 1. Tải File (ASEImporter::InternReadFile)

```
Đầu vào: Đường dẫn file và IOSystem
↓
Mở và đọc file vào bộ đệm bộ nhớ
↓
Xác định phiên bản định dạng file (110 cho .asc, 200 cho .ase)
↓
Tạo instance ASE::Parser
```

**Vị trí**: `code/AssetLib/ASE/ASELoader.cpp:115`

**Các bước chính**:
- Mở file `.ase` bằng IOSystem
- Đọc toàn bộ nội dung file vào bộ đệm bộ nhớ
- Xác định phiên bản định dạng dựa trên phần mở rộng file (ASC = v110, ASE = v200)
- Tạo instance parser với buffer và phiên bản định dạng

### 2. Giai Đoạn Phân Tích (Parser::Parse)

```
Khởi tạo Parser
↓
Vòng lặp phân tích chính - xử lý các token
├── *3DSMAX_ASCIIEXPORT → Đọc phiên bản định dạng
├── *SCENE → ParseLV1SceneBlock()
├── *MATERIAL_LIST → ParseLV1MaterialListBlock()
├── *GEOMOBJECT → ParseLV1ObjectBlock() cho meshes
├── *LIGHTOBJECT → ParseLV1ObjectBlock() cho lights
├── *CAMERAOBJECT → ParseLV1ObjectBlock() cho cameras
└── *HELPEROBJECT → ParseLV1ObjectBlock() cho dummies
↓
Dữ liệu đã phân tích được lưu trong các vector thành viên của Parser
```

**Vị trí**: `code/AssetLib/ASE/ASEParser.cpp:231`

**Phân cấp phân tích**:

- **Level 1 (LV1)**: Các khối cấp cao nhất
  - `ParseLV1SceneBlock()`: Cài đặt scene (phạm vi frame, tốc độ, nền)
  - `ParseLV1MaterialListBlock()`: Tất cả các vật liệu trong scene
  - `ParseLV1ObjectBlock()`: Các đối tượng riêng lẻ (meshes, lights, cameras, dummies)

- **Level 2 (LV2)**: Các khối cấp độ đối tượng
  - `ParseLV2MaterialBlock()`: Thuộc tính vật liệu riêng lẻ
  - `ParseLV2NodeTransformBlock()`: Ma trận biến đổi node
  - `ParseLV2AnimationBlock()`: Dữ liệu animation
  - `ParseLV2MeshBlock()`: Dữ liệu hình học mesh
  - `ParseLV2LightSettingsBlock()`: Cài đặt ánh sáng
  - `ParseLV2CameraSettingsBlock()`: Cài đặt camera

- **Level 3 (LV3)**: Các khối dữ liệu chi tiết
  - `ParseLV3MeshVertexListBlock()`: Vị trí vertex
  - `ParseLV3MeshFaceListBlock()`: Chỉ số face
  - `ParseLV3MeshTListBlock()`: Tọa độ texture
  - `ParseLV3MeshTFaceListBlock()`: Chỉ số face texture
  - `ParseLV3MeshCListBlock()`: Màu vertex
  - `ParseLV3MeshNormalListBlock()`: Normals của vertex
  - `ParseLV3MapBlock()`: Texture maps
  - Các khối animation: Keyframes vị trí, xoay, tỉ lệ

- **Level 4 (LV4)**: Phân tích dữ liệu nguyên thủy
  - `ParseLV4MeshRealTriple()`: Phân tích 3 giá trị float (vị trí, normals)
  - `ParseLV4MeshLongTriple()`: Phân tích 3 giá trị integer (chỉ số face)
  - `ParseLV4MeshFace()`: Phân tích dữ liệu face riêng lẻ

### 3. Chuyển Đổi Dữ Liệu (ASEImporter::InternReadFile tiếp tục)

Sau khi phân tích hoàn tất, dữ liệu được chuyển đổi sang định dạng nội bộ của Assimp:

```
Dữ liệu ASE đã phân tích
↓
Với mỗi mesh:
├── BuildUniqueRepresentation() - Tạo các vertex độc nhất
├── GenerateNormals() - Tạo/xác thực normals
└── ConvertMeshes() - Chuyển đổi sang đối tượng aiMesh
↓
BuildMaterialIndices() - Thiết lập tham chiếu vật liệu
↓
Thu thập tất cả các nodes (meshes, lights, cameras, dummies)
↓
BuildNodes() - Tạo phân cấp scene
↓
BuildAnimations() - Thiết lập các kênh animation
↓
BuildCameras() - Tạo đối tượng camera
↓
BuildLights() - Tạo đối tượng ánh sáng
↓
Cấu trúc aiScene hoàn chỉnh
```

**Vị trí**: `code/AssetLib/ASE/ASELoader.cpp:150-245`

#### 3.1 Xử Lý Mesh

**BuildUniqueRepresentation()**: 
- Loại bỏ các vertex trùng lặp
- Tạo chỉ số vertex/normal/UV/màu thống nhất
- Đảm bảo mỗi vertex là độc nhất trong mesh cuối cùng

**GenerateNormals()**:
- Kiểm tra xem normals có tồn tại trong file không
- Tạo normals mượt dựa trên smoothing groups nếu cần
- Có thể được cấu hình để luôn tính toán lại normals thông qua `AI_CONFIG_IMPORT_ASE_RECONSTRUCT_NORMALS`

**ConvertMeshes()**:
- Chia mesh theo vật liệu (một aiMesh cho mỗi vật liệu)
- Tạo vertex buffers với vị trí, normals, UVs, màu sắc
- Thiết lập chỉ số face
- Gán chỉ số vật liệu

#### 3.2 Xây Dựng Scene Graph

**BuildNodes()**:
- Tạo cấu trúc node phân cấp
- Xử lý mối quan hệ cha-con
- Áp dụng ma trận biến đổi
- Gắn meshes vào nodes

**BuildAnimations()**:
- Tạo đối tượng aiAnimation
- Thiết lập các kênh animation cho mỗi node động
- Chuyển đổi dữ liệu keyframe (vị trí, xoay, tỉ lệ)
- Xử lý target animations cho lights và cameras

#### 3.3 Tạo Asset

**BuildMaterialIndices()**:
- Giải quyết tham chiếu vật liệu
- Xử lý submaterials
- Tạo danh sách vật liệu cuối cùng

**BuildCameras()**:
- Chuyển đổi cameras ASE sang đối tượng aiCamera
- Thiết lập FOV, near/far planes
- Xử lý target cameras

**BuildLights()**:
- Chuyển đổi lights ASE sang đối tượng aiLight
- Thiết lập loại ánh sáng (omni, directional, spot)
- Áp dụng màu và cường độ

### 4. Đầu Ra

Đầu ra cuối cùng là một cấu trúc `aiScene` hoàn chỉnh chứa:
- Phân cấp scene (nodes)
- Meshes với dữ liệu hình học
- Vật liệu với textures
- Ánh sáng
- Cameras
- Animations

## Các Tính Năng Chính

### Các Phần Tử ASE Được Hỗ Trợ

✓ **Hình Học**
- Vị trí vertex
- Định nghĩa face
- Normals (với smoothing groups)
- Nhiều kênh UV (lên đến `AI_MAX_NUMBER_OF_TEXTURECOORDS`)
- Màu vertex
- Chỉ số vật liệu trên mỗi face

✓ **Vật Liệu**
- Màu diffuse, specular, ambient
- Độ trong suốt
- Độ bóng
- Texture maps (diffuse, specular, bump, v.v.)
- Submaterials (hỗ trợ multi-material)

✓ **Các Phần Tử Scene**
- Ánh sáng (omni, target, directional, free)
- Cameras (free, target)
- Đối tượng dummy (helpers)
- Phân cấp scene với các phép biến đổi

✓ **Animation**
- Keyframes vị trí
- Keyframes xoay
- Keyframes tỉ lệ
- Nhiều loại animation (Track, Bezier, TCB)
- Target animations cho lights/cameras

✓ **Skinning** (hỗ trợ một phần)
- Định nghĩa bone
- Vertex weights

### Các Phiên Bản Định Dạng File

- **Version 110**: Định dạng cũ (phần mở rộng `.asc`)
- **Version 200**: Định dạng hiện tại (phần mở rộng `.ase`)

Parser tự động phát hiện và xử lý cả hai phiên bản.

## Các Tùy Chọn Cấu Hình

- `AI_CONFIG_IMPORT_ASE_RECONSTRUCT_NORMALS` (mặc định: 1)
  - Khi được bật, luôn tính toán lại normals bất kể nội dung file
  - Giải pháp hữu ích cho các ASE normals bị lỗi từ 3DS Max

- `AI_CONFIG_IMPORT_NO_SKELETON_MESHES` (mặc định: 0)
  - Kiểm soát việc có tạo meshes hình ảnh hóa skeleton hay không

## Xử Lý Lỗi

Parser bao gồm xử lý lỗi mạnh mẽ:
- Xác thực cấu trúc file và chuỗi token
- Kiểm tra EOF không mong đợi
- Cảnh báo về sự không nhất quán (số lượng vertex/face)
- Xác thực dấu ngoặc khớp nhau
- Theo dõi số dòng để báo cáo lỗi

## Chi Tiết Triển Khai

### Phân Tích Token

Parser sử dụng phương pháp đơn giản dựa trên token:
- `SkipToNextToken()`: Tiến đến ký tự `*`, `{`, hoặc `}` tiếp theo
- `SkipSection()`: Bỏ qua toàn bộ khối bao gồm các phần con lồng nhau
- `TokenMatch()`: Kiểm tra xem vị trí hiện tại có khớp với từ khóa không
- Macros `AI_ASE_HANDLE_TOP_LEVEL_SECTION()` và `AI_ASE_HANDLE_SECTION()` quản lý độ sâu khối

### Quản Lý Bộ Nhớ

- File được tải hoàn toàn vào bộ nhớ để truy cập ngẫu nhiên nhanh
- Parser duy trì con trỏ đến vị trí hiện tại trong buffer
- Không có cấp phát động trong quá trình phân tích (sử dụng vectors được cấp phát trước)
- Xử lý chuỗi hiệu quả bằng con trỏ kiểu C

### Tối Ưu Hóa

- Phân tích một lượt
- Sao chép chuỗi tối thiểu
- Truy cập buffer bộ nhớ trực tiếp
- Cấu trúc dữ liệu được cấp phát trước

## Kiểm Thử

Các file test nằm ở:
- `test/models/ASE/` - Các file test hợp lệ
- `test/models-nonbsd/ASE/` - Các file test bổ sung
- `test/models/invalid/` - Các trường hợp test không hợp lệ

Phạm vi kiểm thử bao gồm:
- Import hình học cơ bản
- Import animation
- Animation camera
- Nhiều vật liệu
- Các file mã hóa UTF-16
- Các trường hợp biên và điều kiện lỗi

## Tổng Kết

Plugin ASE của Assimp cung cấp một cách mạnh mẽ và hiệu quả để import các file 3D từ 3ds Max. Quy trình làm việc của nó bao gồm:

1. **Đọc file** vào bộ nhớ
2. **Phân tích** cú pháp ASCII theo cấu trúc phân cấp
3. **Chuyển đổi** dữ liệu sang cấu trúc scene nội bộ
4. **Xây dựng** scene graph, animations, và assets cuối cùng

Kiến trúc module hóa giúp dễ dàng bảo trì và mở rộng, trong khi việc xử lý lỗi mạnh mẽ đảm bảo parsing đáng tin cậy ngay cả với các file có định dạng không chuẩn.
