# ASE Importer Documentation

This directory contains comprehensive documentation for the ASE (ASCII Scene Export) file format importer in Assimp.

## Documentation Files

### 1. [ASE Workflow Documentation (ASE_WORKFLOW.md)](ASE_WORKFLOW.md)
**Comprehensive guide to the ASE file parsing workflow**

This document provides an in-depth explanation of how Assimp processes ASE files, including:
- Complete architecture overview
- File format specification and structure
- Data structures used in parsing
- Step-by-step workflow from file loading to scene generation
- Visual workflow diagram
- Supported features and limitations
- Configuration options
- Error handling
- Implementation details

**Also available in Vietnamese** - The document includes a complete Vietnamese translation for developers who prefer to read in Vietnamese.

### 2. [ASE Usage Examples (ASE_USAGE_EXAMPLES.md)](ASE_USAGE_EXAMPLES.md)
**Practical code examples for using the ASE importer**

This document contains ready-to-use code examples demonstrating:
- Basic file loading
- Processing mesh data (vertices, faces, normals, UVs, colors)
- Extracting materials and textures
- Processing animations
- Working with lights and cameras
- Configuration options
- Complete application example
- Common post-processing flags
- Tips and best practices
- Compilation and linking instructions

## Quick Start

To load an ASE file in your application:

```cpp
#include <assimp/Importer.hpp>
#include <assimp/scene.h>
#include <assimp/postprocess.h>

Assimp::Importer importer;
const aiScene* scene = importer.ReadFile("model.ase", 
    aiProcess_Triangulate | 
    aiProcess_GenSmoothNormals | 
    aiProcess_ValidateDataStructure);

if (!scene) {
    // Handle error
    std::cerr << importer.GetErrorString() << std::endl;
}
```

For complete examples, see [ASE_USAGE_EXAMPLES.md](ASE_USAGE_EXAMPLES.md).

## About ASE Files

ASE (ASCII Scene Export) is a text-based 3D scene format exported from Autodesk 3ds Max. Key characteristics:

- **Format**: ASCII text (human-readable)
- **File Extensions**: `.ase`, `.ask`, `.asc`
- **Origin**: 3ds Max ASCII export format
- **Versions**: 110 (older .asc), 200 (current .ase)

### Supported Features

✓ Geometry (vertices, faces, normals, UVs, vertex colors)  
✓ Materials and textures  
✓ Scene hierarchy  
✓ Lights (omni, directional, spot)  
✓ Cameras  
✓ Animations (keyframe-based)  
✓ Smoothing groups  
✓ Multiple UV channels  

## Configuration

Key configuration options for ASE import:

- `AI_CONFIG_IMPORT_ASE_RECONSTRUCT_NORMALS` (default: 1)
  - Force recompute normals (workaround for 3DS Max broken normals)

- `AI_CONFIG_IMPORT_NO_SKELETON_MESHES` (default: 0)
  - Control skeleton mesh generation for animations

## Common Issues and Solutions

### Problem: Incorrect Normals
**Solution**: Set `AI_CONFIG_IMPORT_ASE_RECONSTRUCT_NORMALS` to 1 to force normal recalculation.

### Problem: Missing Textures
**Solution**: Ensure texture paths in ASE file are correct relative to the ASE file location. You may need to resolve paths based on your asset directory structure.

### Problem: Scene Appears Empty
**Solution**: Always traverse the scene graph recursively starting from `scene->mRootNode`. ASE files can have complex hierarchies.

## Testing

Test files are available in:
- `test/models/ASE/` - Valid test files
- `test/models-nonbsd/ASE/` - Additional test files (with different licenses)
- `test/models/invalid/` - Invalid test cases

Run tests with:
```bash
ctest -R ASE
```

## Related Documentation

- [Main Assimp Documentation](../Readme.md)
- [Supported File Formats](Fileformats.md)
- [Data Structures](datastructure.xml)
- [Architecture Overview](architecture/)

## Contributing

If you find issues with the ASE importer or have suggestions for improvements:
1. Check existing issues on GitHub
2. Create a new issue with detailed information
3. Include sample ASE files if possible
4. Follow the [contribution guidelines](../CONTRIBUTING.md)

## Source Code

The ASE importer source code is located at:
- Parser: `code/AssetLib/ASE/ASEParser.h` and `ASEParser.cpp`
- Importer: `code/AssetLib/ASE/ASELoader.h` and `ASELoader.cpp`

## License

The Assimp library and its documentation are licensed under the terms of the 3-clause BSD license. See [LICENSE](../LICENSE) for details.

---

**Last Updated**: February 2026  
**Assimp Version**: 5.x+
