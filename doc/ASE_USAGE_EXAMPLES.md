# ASE Importer Usage Examples

This document provides practical examples of using the ASE importer in Assimp.

## Basic Usage

### Example 1: Loading an ASE File

```cpp
#include <assimp/Importer.hpp>
#include <assimp/scene.h>
#include <assimp/postprocess.h>

int main() {
    // Create an importer instance
    Assimp::Importer importer;
    
    // Load the ASE file
    const aiScene* scene = importer.ReadFile("path/to/model.ase", 
        aiProcess_Triangulate |           // Convert polygons to triangles
        aiProcess_JoinIdenticalVertices | // Optimize mesh
        aiProcess_GenSmoothNormals |      // Generate smooth normals
        aiProcess_ValidateDataStructure   // Validate the imported data
    );
    
    // Check if import was successful
    if (!scene || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode) {
        std::cerr << "Error loading ASE file: " << importer.GetErrorString() << std::endl;
        return -1;
    }
    
    // Successfully loaded!
    std::cout << "Loaded ASE file successfully!" << std::endl;
    std::cout << "Number of meshes: " << scene->mNumMeshes << std::endl;
    std::cout << "Number of materials: " << scene->mNumMaterials << std::endl;
    
    return 0;
}
```

### Example 2: Processing Mesh Data

```cpp
void ProcessMesh(const aiMesh* mesh, const aiScene* scene) {
    std::cout << "Processing mesh: " << mesh->mName.C_Str() << std::endl;
    std::cout << "  Vertices: " << mesh->mNumVertices << std::endl;
    std::cout << "  Faces: " << mesh->mNumFaces << std::endl;
    
    // Process vertices
    for (unsigned int i = 0; i < mesh->mNumVertices; i++) {
        aiVector3D vertex = mesh->mVertices[i];
        
        // Get normal if available
        if (mesh->HasNormals()) {
            aiVector3D normal = mesh->mNormals[i];
            // Use vertex and normal...
        }
        
        // Get texture coordinates if available
        if (mesh->HasTextureCoords(0)) {
            aiVector3D uv = mesh->mTextureCoords[0][i];
            // Use UV coordinates...
        }
        
        // Get vertex colors if available
        if (mesh->HasVertexColors(0)) {
            aiColor4D color = mesh->mColors[0][i];
            // Use vertex color...
        }
    }
    
    // Process faces
    for (unsigned int i = 0; i < mesh->mNumFaces; i++) {
        const aiFace& face = mesh->mFaces[i];
        // Process face indices...
        for (unsigned int j = 0; j < face.mNumIndices; j++) {
            unsigned int index = face.mIndices[j];
            // Use index...
        }
    }
}

void ProcessNode(const aiNode* node, const aiScene* scene) {
    // Process all meshes in this node
    for (unsigned int i = 0; i < node->mNumMeshes; i++) {
        const aiMesh* mesh = scene->mMeshes[node->mMeshes[i]];
        ProcessMesh(mesh, scene);
    }
    
    // Recursively process all child nodes
    for (unsigned int i = 0; i < node->mNumChildren; i++) {
        ProcessNode(node->mChildren[i], scene);
    }
}
```

### Example 3: Extracting Materials

```cpp
void ProcessMaterial(const aiMaterial* material) {
    aiString name;
    material->Get(AI_MATKEY_NAME, name);
    std::cout << "Material: " << name.C_Str() << std::endl;
    
    // Get diffuse color
    aiColor3D diffuseColor(0.f, 0.f, 0.f);
    material->Get(AI_MATKEY_COLOR_DIFFUSE, diffuseColor);
    std::cout << "  Diffuse: (" << diffuseColor.r << ", " 
              << diffuseColor.g << ", " << diffuseColor.b << ")" << std::endl;
    
    // Get specular color
    aiColor3D specularColor(0.f, 0.f, 0.f);
    material->Get(AI_MATKEY_COLOR_SPECULAR, specularColor);
    
    // Get ambient color
    aiColor3D ambientColor(0.f, 0.f, 0.f);
    material->Get(AI_MATKEY_COLOR_AMBIENT, ambientColor);
    
    // Get shininess
    float shininess = 0.0f;
    material->Get(AI_MATKEY_SHININESS, shininess);
    
    // Get transparency
    float opacity = 1.0f;
    material->Get(AI_MATKEY_OPACITY, opacity);
    
    // Check for diffuse texture
    if (material->GetTextureCount(aiTextureType_DIFFUSE) > 0) {
        aiString texturePath;
        material->GetTexture(aiTextureType_DIFFUSE, 0, &texturePath);
        std::cout << "  Diffuse texture: " << texturePath.C_Str() << std::endl;
    }
    
    // Check for other texture types
    if (material->GetTextureCount(aiTextureType_SPECULAR) > 0) {
        aiString texturePath;
        material->GetTexture(aiTextureType_SPECULAR, 0, &texturePath);
        std::cout << "  Specular texture: " << texturePath.C_Str() << std::endl;
    }
    
    if (material->GetTextureCount(aiTextureType_HEIGHT) > 0) {
        aiString texturePath;
        material->GetTexture(aiTextureType_HEIGHT, 0, &texturePath);
        std::cout << "  Bump map: " << texturePath.C_Str() << std::endl;
    }
}
```

### Example 4: Processing Animations

```cpp
void ProcessAnimations(const aiScene* scene) {
    if (!scene->HasAnimations()) {
        std::cout << "No animations found." << std::endl;
        return;
    }
    
    std::cout << "Number of animations: " << scene->mNumAnimations << std::endl;
    
    for (unsigned int i = 0; i < scene->mNumAnimations; i++) {
        const aiAnimation* animation = scene->mAnimations[i];
        
        std::cout << "Animation " << i << ": " << animation->mName.C_Str() << std::endl;
        std::cout << "  Duration: " << animation->mDuration << " ticks" << std::endl;
        std::cout << "  Ticks per second: " << animation->mTicksPerSecond << std::endl;
        std::cout << "  Channels: " << animation->mNumChannels << std::endl;
        
        // Process animation channels
        for (unsigned int j = 0; j < animation->mNumChannels; j++) {
            const aiNodeAnim* channel = animation->mChannels[j];
            
            std::cout << "  Channel " << j << ": " << channel->mNodeName.C_Str() << std::endl;
            std::cout << "    Position keys: " << channel->mNumPositionKeys << std::endl;
            std::cout << "    Rotation keys: " << channel->mNumRotationKeys << std::endl;
            std::cout << "    Scaling keys: " << channel->mNumScalingKeys << std::endl;
            
            // Example: Access position keyframes
            for (unsigned int k = 0; k < channel->mNumPositionKeys; k++) {
                const aiVectorKey& key = channel->mPositionKeys[k];
                double time = key.mTime;
                aiVector3D position = key.mValue;
                // Use keyframe data...
            }
            
            // Example: Access rotation keyframes
            for (unsigned int k = 0; k < channel->mNumRotationKeys; k++) {
                const aiQuatKey& key = channel->mRotationKeys[k];
                double time = key.mTime;
                aiQuaternion rotation = key.mValue;
                // Use keyframe data...
            }
            
            // Example: Access scaling keyframes
            for (unsigned int k = 0; k < channel->mNumScalingKeys; k++) {
                const aiVectorKey& key = channel->mScalingKeys[k];
                double time = key.mTime;
                aiVector3D scaling = key.mValue;
                // Use keyframe data...
            }
        }
    }
}
```

### Example 5: Processing Lights

```cpp
void ProcessLights(const aiScene* scene) {
    if (!scene->HasLights()) {
        std::cout << "No lights found." << std::endl;
        return;
    }
    
    std::cout << "Number of lights: " << scene->mNumLights << std::endl;
    
    for (unsigned int i = 0; i < scene->mNumLights; i++) {
        const aiLight* light = scene->mLights[i];
        
        std::cout << "Light " << i << ": " << light->mName.C_Str() << std::endl;
        
        // Light type
        const char* typeStr = "Unknown";
        switch (light->mType) {
            case aiLightSource_DIRECTIONAL: typeStr = "Directional"; break;
            case aiLightSource_POINT: typeStr = "Point (Omni)"; break;
            case aiLightSource_SPOT: typeStr = "Spot"; break;
            case aiLightSource_AMBIENT: typeStr = "Ambient"; break;
            case aiLightSource_AREA: typeStr = "Area"; break;
        }
        std::cout << "  Type: " << typeStr << std::endl;
        
        // Position
        std::cout << "  Position: (" << light->mPosition.x << ", " 
                  << light->mPosition.y << ", " << light->mPosition.z << ")" << std::endl;
        
        // Direction (for directional and spot lights)
        if (light->mType == aiLightSource_DIRECTIONAL || light->mType == aiLightSource_SPOT) {
            std::cout << "  Direction: (" << light->mDirection.x << ", " 
                      << light->mDirection.y << ", " << light->mDirection.z << ")" << std::endl;
        }
        
        // Color
        std::cout << "  Diffuse: (" << light->mColorDiffuse.r << ", " 
                  << light->mColorDiffuse.g << ", " << light->mColorDiffuse.b << ")" << std::endl;
        std::cout << "  Specular: (" << light->mColorSpecular.r << ", " 
                  << light->mColorSpecular.g << ", " << light->mColorSpecular.b << ")" << std::endl;
        std::cout << "  Ambient: (" << light->mColorAmbient.r << ", " 
                  << light->mColorAmbient.g << ", " << light->mColorAmbient.b << ")" << std::endl;
        
        // Spot light parameters
        if (light->mType == aiLightSource_SPOT) {
            std::cout << "  Inner cone angle: " << light->mAngleInnerCone << " rad" << std::endl;
            std::cout << "  Outer cone angle: " << light->mAngleOuterCone << " rad" << std::endl;
        }
        
        // Attenuation
        std::cout << "  Attenuation (constant): " << light->mAttenuationConstant << std::endl;
        std::cout << "  Attenuation (linear): " << light->mAttenuationLinear << std::endl;
        std::cout << "  Attenuation (quadratic): " << light->mAttenuationQuadratic << std::endl;
    }
}
```

### Example 6: Processing Cameras

```cpp
void ProcessCameras(const aiScene* scene) {
    if (!scene->HasCameras()) {
        std::cout << "No cameras found." << std::endl;
        return;
    }
    
    std::cout << "Number of cameras: " << scene->mNumCameras << std::endl;
    
    for (unsigned int i = 0; i < scene->mNumCameras; i++) {
        const aiCamera* camera = scene->mCameras[i];
        
        std::cout << "Camera " << i << ": " << camera->mName.C_Str() << std::endl;
        
        // Position
        std::cout << "  Position: (" << camera->mPosition.x << ", " 
                  << camera->mPosition.y << ", " << camera->mPosition.z << ")" << std::endl;
        
        // Look-at direction
        std::cout << "  Look at: (" << camera->mLookAt.x << ", " 
                  << camera->mLookAt.y << ", " << camera->mLookAt.z << ")" << std::endl;
        
        // Up vector
        std::cout << "  Up: (" << camera->mUp.x << ", " 
                  << camera->mUp.y << ", " << camera->mUp.z << ")" << std::endl;
        
        // Field of view
        std::cout << "  Horizontal FOV: " << camera->mHorizontalFOV << " rad" << std::endl;
        
        // Clip planes
        std::cout << "  Near plane: " << camera->mClipPlaneNear << std::endl;
        std::cout << "  Far plane: " << camera->mClipPlaneFar << std::endl;
        
        // Aspect ratio
        std::cout << "  Aspect ratio: " << camera->mAspect << std::endl;
    }
}
```

### Example 7: Configuration Options

```cpp
#include <assimp/Importer.hpp>
#include <assimp/config.h>

void LoadASEWithCustomSettings() {
    Assimp::Importer importer;
    
    // Always reconstruct normals (workaround for 3DS Max broken normals)
    importer.SetPropertyInteger(AI_CONFIG_IMPORT_ASE_RECONSTRUCT_NORMALS, 1);
    
    // Don't generate skeleton meshes for animations
    importer.SetPropertyInteger(AI_CONFIG_IMPORT_NO_SKELETON_MESHES, 1);
    
    // Set maximum bone weights per vertex
    importer.SetPropertyInteger(AI_CONFIG_PP_LBW_MAX_WEIGHTS, 4);
    
    // Load the file
    const aiScene* scene = importer.ReadFile("model.ase",
        aiProcess_Triangulate |
        aiProcess_JoinIdenticalVertices |
        aiProcess_GenSmoothNormals |
        aiProcess_LimitBoneWeights |
        aiProcess_ValidateDataStructure
    );
    
    if (!scene) {
        std::cerr << "Error: " << importer.GetErrorString() << std::endl;
        return;
    }
    
    // Process scene...
}
```

### Example 8: Complete Application

```cpp
#include <assimp/Importer.hpp>
#include <assimp/scene.h>
#include <assimp/postprocess.h>
#include <iostream>

class ASEModelLoader {
public:
    bool LoadModel(const std::string& filepath) {
        Assimp::Importer importer;
        
        // Configure importer
        importer.SetPropertyInteger(AI_CONFIG_IMPORT_ASE_RECONSTRUCT_NORMALS, 1);
        
        // Load file with post-processing
        scene = importer.ReadFile(filepath,
            aiProcess_Triangulate |
            aiProcess_GenSmoothNormals |
            aiProcess_FlipUVs |
            aiProcess_JoinIdenticalVertices |
            aiProcess_ValidateDataStructure
        );
        
        if (!scene || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode) {
            std::cerr << "Failed to load model: " << importer.GetErrorString() << std::endl;
            return false;
        }
        
        // Store the importer to keep the scene alive
        this->importer = std::move(importer);
        
        return true;
    }
    
    void PrintSceneInfo() {
        if (!scene) {
            std::cout << "No scene loaded." << std::endl;
            return;
        }
        
        std::cout << "=== Scene Information ===" << std::endl;
        std::cout << "Meshes: " << scene->mNumMeshes << std::endl;
        std::cout << "Materials: " << scene->mNumMaterials << std::endl;
        std::cout << "Animations: " << scene->mNumAnimations << std::endl;
        std::cout << "Lights: " << scene->mNumLights << std::endl;
        std::cout << "Cameras: " << scene->mNumCameras << std::endl;
        std::cout << "Textures: " << scene->mNumTextures << std::endl;
        std::cout << std::endl;
        
        // Print mesh details
        for (unsigned int i = 0; i < scene->mNumMeshes; i++) {
            const aiMesh* mesh = scene->mMeshes[i];
            std::cout << "Mesh " << i << ": " << mesh->mName.C_Str() << std::endl;
            std::cout << "  Vertices: " << mesh->mNumVertices << std::endl;
            std::cout << "  Faces: " << mesh->mNumFaces << std::endl;
            std::cout << "  Material index: " << mesh->mMaterialIndex << std::endl;
            std::cout << "  Has normals: " << (mesh->HasNormals() ? "Yes" : "No") << std::endl;
            std::cout << "  Has texture coords: " << (mesh->HasTextureCoords(0) ? "Yes" : "No") << std::endl;
            std::cout << "  Has vertex colors: " << (mesh->HasVertexColors(0) ? "Yes" : "No") << std::endl;
        }
    }
    
    const aiScene* GetScene() const { return scene; }
    
private:
    Assimp::Importer importer;
    const aiScene* scene = nullptr;
};

int main(int argc, char** argv) {
    if (argc < 2) {
        std::cout << "Usage: " << argv[0] << " <ase_file>" << std::endl;
        return 1;
    }
    
    ASEModelLoader loader;
    
    if (!loader.LoadModel(argv[1])) {
        return 1;
    }
    
    loader.PrintSceneInfo();
    
    return 0;
}
```

## Common Post-Processing Flags for ASE Files

When loading ASE files, these post-processing flags are commonly used:

- **aiProcess_Triangulate**: Convert all polygons to triangles (recommended)
- **aiProcess_GenSmoothNormals**: Generate smooth normals if not present
- **aiProcess_JoinIdenticalVertices**: Optimize by joining identical vertices
- **aiProcess_ValidateDataStructure**: Validate the imported data structure
- **aiProcess_FlipUVs**: Flip UV coordinates if needed for your rendering system
- **aiProcess_LimitBoneWeights**: Limit bone weights per vertex (useful for skinned meshes)
- **aiProcess_ImproveCacheLocality**: Reorder vertices for better GPU cache performance
- **aiProcess_RemoveRedundantMaterials**: Remove redundant materials
- **aiProcess_OptimizeMeshes**: Reduce mesh count by merging compatible meshes

## Tips and Best Practices

1. **Always check for null**: Check if `scene`, `scene->mRootNode`, and other pointers are valid before using them.

2. **Handle normals carefully**: ASE files from 3DS Max sometimes have incorrect normals. Use `AI_CONFIG_IMPORT_ASE_RECONSTRUCT_NORMALS` to recompute them.

3. **Process the scene graph recursively**: ASE files can have complex hierarchies. Always traverse the scene graph recursively starting from `scene->mRootNode`.

4. **Keep the importer alive**: The `aiScene` pointer is only valid as long as the `Assimp::Importer` object is alive. Store the importer instance if you need to keep the scene.

5. **Use appropriate post-processing**: Apply post-processing flags based on your needs. For example, use `aiProcess_Triangulate` if your renderer only supports triangles.

6. **Check animation data**: Not all ASE files contain animations. Always check `scene->HasAnimations()` before processing animation data.

7. **Material textures**: Texture paths in ASE files are relative to the ASE file location. You may need to resolve these paths based on your asset directory structure.

8. **Error handling**: Always check the return value of `ReadFile()` and use `GetErrorString()` to get detailed error information.

## Compiling and Linking

To use these examples, compile with:

```bash
g++ -o ase_loader main.cpp -lassimp
```

Or with CMake:

```cmake
find_package(assimp REQUIRED)
target_link_libraries(your_target assimp::assimp)
```
