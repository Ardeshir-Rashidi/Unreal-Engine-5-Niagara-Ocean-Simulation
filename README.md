<h1 align="center">
Ocean Simulation
</h1>

<p align="center">
"This file was developed in Unreal Engine 5.4 and performs best on this version."
</p>

<p align="center">
  <img src="https://github.com/Ardeshir-Rashidi/Unreal-Engine-5-Ocean-Simulation/blob/7751cda91737f3bdbbaa0ccc402e425ac2b37cad/OceanSimulation/Cover.png" alt="Ocean Simulation Unreal Engine 5">
</p>

<br/>

### Video Preview:
> https://www.youtube.com/watch?v=Xhm2Syyu3fo

<br/>

> [!NOTE]
> The `Guide` folder contains images for the `README.md` file only. You can delete this folder after downloading the files and use the `OceanSimulation` folder to launch the project.

<br/>

## [Overview]()

Rendering a realistic ocean surface has been a longstanding challenge in real-time rendering. While the concept is simple - summing a few wave signals - the reality requires handling thousands of wave signals simultaneously.

Traditionally, ocean surfaces were created by panning textures across a surface, a technique that, while still viable, lacks realism. As hardware evolved, the advent of flexible, programmable vertex, and pixel shaders enabled the evaluation of wave sums directly in shaders. However, even today, processing more than 30 wave signals directly in a shader is impractical.

This challenge spurred the development of various techniques, one of which involves rendering waves into a texture every frame and then using that texture in the water shader. Although this method improves performance, it doesn't fully capture the complexity of thousands of waves. The solution lies in generating a texture every frame where each texel represents a discrete ocean height. For example, a modest 256x256 texture with 30 waves requires evaluating a sine function 1,966,080 times per frame - a significant load, even for a GPU.

The key to optimizing this process is recognizing that the waves need not be entirely arbitrary. For instance, a sine wave traveling horizontally across the texture only requires 256 evaluations. If another wave has half the wavelength, many points can be reused between the two waves. This repetition on a discrete grid forms the basis for speeding up the process.

The Fast Fourier Transform (FFT) leverages this property. Without delving into the math, it's essential to understand that any signal can be decomposed into sines and cosines. In our case, the height texture represents a spatial domain—height across space. However, this data can also be represented as a set of amplitudes and phases varying over frequency - known as the frequency domain.

Fourier Transform converts between these domains: forward transform from spatial to frequency domain, and inverse transform from frequency to spatial. The Fast Fourier Transform (FFT) is an optimized version of this operation.

With this understanding, we define our water heightfield in the Fourier domain, where amplitude and phase vary with frequency. Amplitude and phase are represented by complex numbers, with the vector's direction indicating phase and its length representing amplitude. Frequency is determined by the distance from the grid's center, and wave direction is similarly defined. Together, frequency and direction form a wavevector, with its direction indicating wave direction and magnitude representing the inverse frequency.

In this tutorial, we'll implement this in Niagara. We'll generate a field of random complex amplitudes, control their distribution, and update the phase of each wave signal each frame based on its frequency and the elapsed time. This process defines our heightfield.

Finally, we'll perform an Inverse Fast Fourier Transform (IFFT) to produce a heightmap, or more precisely, a displacement map. We'll process these maps for use in shaders and create a basic material for previewing the results. We'll also generate multiple textures at different scales to combine in the material.

We'll conclude by exploring how different parameter values affect the simulation and discuss potential improvements.

Now, let's move on to the next step: creating and setting up the Niagara system.

<br/>

## [Parameter Exploration]()

<p align="center">
  <img src="https://github.com/Ardeshir-Rashidi/Unreal-Engine-5-Ocean-Simulation/blob/main/Guide/PercascadeParams.png?raw=true">
</p>

- #### WindDirectionality
At a value of 1, **WindDirectionality** eliminates waves traveling upwind, while at 0, it balances upwind and downwind waves. A value around 0.6 works well for most cases.

- #### Amplitude
While the **Amplitude** parameter doesn't directly correlate to a specific value in the spectrum, it's important to increase cascade amplitude by the square of the ratio between the current and larger cascade PatchLength for realism. Emphasizing smaller cascades is key here.

- #### Choppiness
For realistic surfaces, set **Choppiness** between 0.7 and 1.5. Don't hesitate to increase it to allow wave crests to overlap for more natural results.

- #### PatchLength
The most critical parameter is **PatchLength**. Ideally, this should be set so that the largest cascade is significantly longer than the longest waves generated by the wind speed used. Since wavelengths and directions are grid-dependent, only a few long waves will be present, with the majority being short waves. The spectrum is sampled with varying density depending on wavelength, so it's crucial to prevent the longest waves from showing up, as they are few but highly impactful, leading to noticeable repetition.

Smaller cascades should have an irrational scale factor relative to the largest one. A good approach is to scale the second-largest cascade by approximately 1.68 times smaller than the largest one, then make a significant jump down to meter scale, followed by a similar ratio for the smaller cascades. This strategy helps eliminate visible tiling.

- #### ShortWaveCutOff
**ShortWaveCutOff** addresses the issue of spatial aliasing from too-short waves, which can lead to inaccurate normal and displacement representations. It also prevents the excessive presence of short waves when wind direction is diagonal. Adjust this parameter so that the spectrum preview looks more circular, reducing the presence of extreme wavelengths at the grid's corners.

- #### LongWaveCutOff
To avoid overlapping cascades, adjust the **LongWaveCutOff** parameter. Enable SpectrumGrid preview and tweak the parameter until a black spot appears at the spectrum center. The goal is to cut about one-fifth of the signals in the center for all cascades except the largest.

- #### WindTighten
Increasing **WindTighten** suppresses waves traveling perpendicular to the wind direction. Values between 3 and 8 provide a good balance.

<hr>

<p align="center">
  <img src="https://github.com/Ardeshir-Rashidi/Unreal-Engine-5-Ocean-Simulation/blob/main/Guide/WindControl.png?raw=true">
</p>

- #### RepeatPeriod
The **RepeatPeriod** parameter manages precision issues that arise during iFFT computation over time. Setting it too high can introduce jitter and noticeable jumps during animation resets, while setting it too low can cause unrealistic wave propagation. A range of 200-1000 is recommended.

<hr>

<p align="center">
  <img src="https://github.com/Ardeshir-Rashidi/Unreal-Engine-5-Ocean-Simulation/blob/main/Guide/PercascadeFoamParams.png?raw=true">
</p>

- #### Foam Parameters
   - **FoamInjection**: Typically set to 1, but can be reduced for less dense foam.
   - **FoamThreshold**: Controls when foam starts injecting as wave crests collapse, related to Choppiness.
   - **FoamBlur**: Adjusts foam blur speed based on frame rate, with a good range being 0.1-20.
   - **FoamFade**: Determines how quickly foam fades from the simulation, with a range of 0.01-0.5.

<hr>

<p align="center">
  <img src="https://github.com/Ardeshir-Rashidi/Unreal-Engine-5-Ocean-Simulation/blob/main/Guide/RoughnessParams.png?raw=true">
</p>

- #### Roughness Parameters
   - **RoughnessPower**: Controls how fast roughness increases with distance, with a range of 0.125 - 8.
   - **NumRoughnessIntegrationSamples**: The number of frames over which roughness integration is performed. Increasing this value enhances accuracy but reduces responsiveness. A range of 64-512 is effective.

<br/>

## [NiagaraOceanSystem: Closing Notes and Further Improvements]()

What has been presented is a rough outline that doesn’t need to be followed strictly. There is significant room for optimization and customization:

   1. **Mesh Optimization**: Use a proper mesh for ocean rendering, whether a quad-tree-based mesh, projected grid, or a simple quad snapped to the camera.
   2. **Cascade Adjustment**: Adjust the number and size of cascades as needed.
   3. **Pass Configuration**: Consider running cascades in separate passes for reduced memory consumption, though this may slow things down.
   4. **Mip Generation**: When Niagara gets mip generation for render target arrays, replace the current method with render target arrays to improve efficiency.

Further enhancements include optimizing pixel attributes, exploring different bit depths for displacement storage, and experimenting with SM6 features to accelerate blur, derivative calculation, and the iFFT pass.

For specular anti-aliasing, there are proven methods like CLEAN mapping, but an alternative is using a roughness look-up table pre-integrated for a range of pixel sizes, which has several advantages over traditional methods.

Finally, consider adding mechanisms to seamlessly change wind speed and direction, which can be done by fading wave signals in and out of the spectrum. Experimentation is encouraged to achieve the most realistic and optimized results.

<br/>
<br/>

> [!WARNING]
> ### How to Use the Sample Files
> If you choose to download the sample files, please note that the Niagara modules include custom HLSL expressions that use absolute paths. This means you'll need to adjust the paths in the relevant modules.
> <br/>
> Shader include files are located in the `Content` folder of the project.
> 
> Here’s how to set up the include paths:
>
> 1. **Open `FX_OceanWater_Timestep` Niagara Module Script:**
>   
>      - Select the `Custom HLSL Expression` node.
>      - In the Details panel, locate the entry for `Absolute Include File Paths`.
>      - Click the `...` button next to this entry.
>      - Browse to `OceanSimulation\Content\OceanSystem\OceanWater` and select the `OceanComplexMath.ush` file.
>     
> 2. **Open `FX_OceanWater_Rowpass` Niagara Module Script:**
>   
>      - Select the `Custom HLSL Expression` node.
>      - In the Details panel, find the `Absolute Include File Paths` entry.
>      - Click the `...` button next to this entry.
>      - Browse to `OceanSimulation\Content\OceanSystem\OceanWater` and select the `OceanWater.ush` file.
>     
> 3. **Open `FX_OceanWater_Colpass` Niagara Module Script:**
>   
>      - Select the `Custom HLSL Expression` node.
>      - In the Details panel, find the `Absolute Include File Paths` entry.
>      - Click the `...` button next to this entry.
>      - Browse to `OceanSimulation\Content\OceanSystem\OceanWater` and select the `OceanWater.ush` file.
>     
> 4. **Open `FX_OceanWater_ExportData` Niagara Module Script:**
>   
>      - Select the `Custom HLSL Expression` node.
>      - In the Details panel, find the `Absolute Include File Paths` entry.
>      - Click the `...` button next to this entry.
>      - Browse to `OceanSimulation\Content\OceanSystem\OceanWater` and select the `OceanExport.ush` file.
>
> <br/>
> After adjusting these paths, the custom HLSL expressions should properly reference the necessary shader files, allowing you to use the sample files effectively in your Niagara setup.

<br/>

<img align="center" src="https://github.com/Ardeshir-Rashidi/Unreal-Engine-5-Ocean-Simulation/blob/main/Guide/Sample%20Files.png?raw=true" alt="Add .ush files to unreal engine 5">
<p align="center">
  How to Use the Sample Files
</p>

<hr>
<br/>

> [!IMPORTANT]
> ### Usage Instructions for the Ocean_Water Material in Different Levels
> Make sure to include the `FX_OceanWater` file located at `OceanSimulation\Content\OceanSystem\OceanWater\Effects` in your levels. If this file is not included, the material will not display animations correctly.
