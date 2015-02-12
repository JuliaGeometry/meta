(skariel) IMO, initially, the TetGen api should follow the C-Api very closely (whatever the C functions require as input, nothing more). Textures, materials, etc. are all subproperties of the geometry. A higher level API should give comfortable types and methods to deal with geometry (vertices, faces and normals) then these can have an aditional field `data` or `info` which is parametric. This would minimaly couple Tetgen with the libs that use it. I guess I'm in the one generic meshtype to rule them all :)


### How should a generic mesh typ look like?
Possible Demands:
* Different combinations of vertex attributes
  1. Color
  2. Normals
  3. Texture Coordinates
  4. Intensities + color map
  5. Materials
  6. Groups
  7. Custom attributes
* Indexlists
  1. One index list indexing into all attributes (attributes must all have the same length)
  2. Different index lists per attribute
  3. Zero / One indexing
* Different Primitives
  1. Polyhedrons
  2. Triangles
  3. Quads
  4. NPolies
  5. Variable Length
  6. Sprites
* Should be able to be compatible with different Formats
  1. Obj, Ply, etc
  2. tetgenio
* Instancing
* 2D/3D
* Volume Meshes

This list includes all kinds of possible demands, so there are items on it, which probably shouldn't go into the meshtype itself.
One challenge is to identify how generic the mesh type should be.

##### Possible tradeoffs:
* One Generic Meshtype to rule them all
  1. less code
  2. same usage pattern accross different domains
  3. interoperability
* Many Specizalized types
  1. possible performance advantages
  2. less awkward for specialized tasks
  
  
First sketch, prototyped in [Meshes2](https://github.com/SimonDanisch/Meshes2.jl)
```Julia
immutable Mesh{Attributes}
    data::Dict{Symbol, Any}
    material::Vector{Vector{RGBA{Float32}}}
    material_id::Vector{Uint32}
    textures::Dict{Symbol, Vector{Matrix}}
    model::Matrix4x4
end
immutable Face{T, N, IndexOffset}
data::NVector{T, N}
end
immutable Vertex{T, N}
data::NVector{T, N}
end
immutable Normal{T, N}
data::NVector{T, N}
end
immutable RGB{T}
data::NVector{T, 3}
end
typealias Triangle{T} Face{T, 3, 1}

typealias GLTriangle Face{Cuint, 3, -1} # Julias 1 is the baseline, so for zero indexing 1 has to be substracted
typealias GLVertex Vertex{Float32, 3}
typealias GLNormal Normal{Float32, 3}
typealias GLRGB RGB{Ufixed8}
N = 300
verts   = GLVertex[... for i=1:N]
faces   = GLTriangle[... for i=1:N]
normals = GLNormal[... for i=1:N]
color   = GLRGB[... for i=1:N]

msh = Mesh(verts, faces, normals, color)
# -> Mesh{( Face{Cuint, 3, 0}, Normal{Float32, 3}, RGB{Ufixed8}, Vertex{Float32, 3})} #parameter list gets sorted

msh[:face] #indexing with lowercase type name, without the parameters
  
```
Nice thing about this design is, that you can state precicely what kind of mesh you need, and then define convert functions which transform the mesh into what you need:
```Julia
msh = Mesh(vertex, faces{4,T})
convert(Mesh{Face, Normal, Vertex}, mesh)  # -> generate normals
convert(Mesh{Triangle, Normal, Vertex}, mesh) #-> triangulates mesh
convert(Mesh{Face, Normal, UV}, mesh) #--> fills in default
```
Question: Is it justified to have all this sorting of parameters and forcing the user to use the right sorting?

Considering one indexing vs zero indexing,one solution could be, to have the index data always start at zero, 
and define indexing methods into julia arrays, which uses the `IndexOffset` offset from the face. 
So you could have something like this:
```Julia
getindex{T, N, Offset}(a::Vector, face::Face{T, N, Offset} = map(x-> a[x+Offset], face) # map from FixedSizeArrays
verts = Vertex[...]
verts[SomeFace] #-> gives Face{T <: Vertex, N}
```
This way, we have c compatibility, without copying that much around.
