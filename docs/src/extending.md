# Extending

There are 3 ways to extend Makie:

1) By creating a new function combining multiple plotting commands (duh)
2) By overloading conversions for your custom type
3) By overloading plot(...) for your own type
4) By adding a new primitive + shader

## Option 1

The first option is quite trivial and can be done in any plotting package and language:
just create a function that scripts together a Plot.

## Option 2

The plotting pipeline heavily relies on conversion functions which check the attributes for validity,
document what's possible to pass and convert them to the types that the backends need.
They usually look like this:

```julia
to_positions(backend, positions) = Point3f0.(positions) # E.g. everything that can be converted to a Point
```

As you can see, the first argument is the backend, so you can overload this for a specific backend
or for a specific position type.
This can look something like this:

```@example to_position
using Makie, GeometryTypes

# To simplify the example, we take the already existing GeometryTypes.Circle type, which
# can already be decomposed into positions
function Makie.to_positions(backend, x::Circle)
    # Convert to a type to_positions can handle.
    # Everything that usually works in e.g. scatter/lines should be allowed here.
    positions = decompose(Point2f0, x, 50)
    # Pass your position data to to_positions,
    # just in case the backend has some extra converts
    # that are not visible in the user facing API.
    Makie.to_positions(backend, positions)
end
scene = Scene(resolution = (500, 500))
p1 = lines(Circle(Point2f0(0), 5f0))
p2 = scatter(Circle(Point2f0(0), 6f0))
center!(scene)
save("ext_plot1.png", scene); nothing # hide
```
![](ext_plot1.png)

since the pipeline for converting attributes also knows about Circle now,
we can update the attribute directly with our own type

```@example to_position
p2[:positions] = Circle(Point2f0(0), 7f0)
center!(scene)
save("ext_plot2.png", scene); nothing # hide
```
![](ext_plot2.png)

## Option 3

Option 2 is very similar to Plots.jl recipes.
Inside the function you can just use all of the plotting and drawing API to create
a rich visual representation of your type.
The signature that needs overloading is:

```julia
function plot(obj::MyType, kw_args::Dict)
    # use primitives and other recipes to create a new plot
    scatter(obj, kw_arg[:my_attribute])
    lines(...)
    polygon(...)
end
```

## Option 3

Option 3 is pretty unique and is a real extension of Makie's functionality as it
adds a new primitive drawing type.
This interface will likely change a lot in the future, since it carries quite a lot of
technical debt from the design of GLAbstraction + GLVisualize, but this is how you can do it right now:

You will need to create defaults for the attributes of your new visual.
For a documentation on how to use this macro look at [Devdocs](@ref).

```julia

my_attribute_convert(A::Array) = Texture(A)
my_attribute_convert(A::Texture) = A
my_attribute_convert(A) = error("A needs to be an array or Texture. Found: $(typeof(A))")

@default function my_drawing_type(scene, kw_args)
    my_attribute = my_attribute_convert(my_attribute)
    another_attribute = to_float(another_attribute) # always gets converted to Float32
end

function my_drawing_type(main_object::MyType, kw_args::Dict)
    # optionally expand keyword arguments. E.g. m = (1, :white) becomes markersize = 1, markercolor = :white
    kw_args = expand_kwargs(kw_args)
    scene = get_global_scene()
    # The default macro adds a _defaults to the function name
    kw_args = my_drawing_type_defaults(scene, kw_args) # fill in and convert attributes

    boundingbox = Signal(AABB(Vec3f0(0), Vec3f0(1))) # calculate a boundingbox from your data

    primitive = GL_TRIANGLES

    vsh = vert"""
        {{GLSL_VERSION}}
        in vec2 position;
        void main(){
            gl_Position = vec4(position, 0, 1.0);
        }
    """

    fsh = frag"""
        {{GLSL_VERSION}}
        out vec4 outColor;
        void main() {
            outColor = vec4(1.0, 1.0, 1.0, 1.0);
        }
    """
    program = LazyShader(vsh, fsh)
    robj = std_renderobject(shader_uniforms, program, boundingbox, primitive, nothing)
    # The attributes that you add to the scene should be all signals and all editable.
    # if an attribute is fixed, don't add it to the scene
    insert_scene!(scene, :scatter, viz, attributes)
    attributes
end

```
