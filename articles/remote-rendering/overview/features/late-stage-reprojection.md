---
title: Late stage reprojection
description: Information on Late Stage Reprojection and how to use it.
author: sebastianpick
ms.author: sepick
ms.date: 02/04/2020
ms.topic: article
---

# Late stage reprojection

*Late Stage Reprojection* (LSR) is a hardware feature that helps stabilize holograms when the user moves.

Static models are expected to visually maintain their position when you move around them. If they appear to be unstable, this behavior may hint at LSR issues. Mind that additional dynamic transformations, like animations or explosion views, might mask this behavior.

You may choose between two different LSR modes, namely **Planar LSR** or **Depth LSR**. Which one is active is determined by whether the client application submits a depth buffer.

Both LSR modes improve hologram stability, although they have their distinct limitations. Start by trying Depth LSR, as it is arguably giving better results in most cases.

## Choose LSR mode in Unity

In the Unity editor, go to *:::no-loc text="File > Build Settings":::*. Select *:::no-loc text="Player Settings":::* in the lower left, then check under *:::no-loc text="Player > XR Settings > Virtual Reality SDKs > Windows Mixed Reality":::* whether **:::no-loc text="Enable Depth Buffer Sharing":::** is checked:

![Depth Buffer Sharing Enabled flag](./media/unity-depth-buffer-sharing-enabled.png)

If it is, your app will use Depth LSR, otherwise it will use Planar LSR.

When using OpenXR, the depth buffer should always be submitted. The setting can be found in *:::no-loc text="XR Plug-in Management > OpenXR":::*. The reprojection mode can then be changed via an extension in the OpenXR plugin:

```cs
using Microsoft.MixedReality.OpenXR;

public class OverrideReprojection : MonoBehaviour
{
    void OnEnable()
    {
        RenderPipelineManager.endCameraRendering += RenderPipelineManager_endCameraRendering;
    }
    void OnDisable()
    {
        RenderPipelineManager.endCameraRendering -= RenderPipelineManager_endCameraRendering;
    }

    // When using the Universal Render Pipeline, OnPostRender has to be called manually.
    private void RenderPipelineManager_endCameraRendering(ScriptableRenderContext context, Camera camera)
    {
        OnPostRender();
    }

    // Called directly when using Unity's legacy renderer.
    private void OnPostRender()
    {
        ReprojectionSettings reprojectionSettings = default;
        reprojectionSettings.ReprojectionMode = ReprojectionMode.PlanarManual; // Or your favorite reprojection mode.
        
        // In case of PlanarManual you also need to provide a focus point here.
        reprojectionSettings.ReprojectionPlaneOverridePosition = ...;
        reprojectionSettings.ReprojectionPlaneOverrideNormal = ...;
        reprojectionSettings.ReprojectionPlaneOverrideVelocity = ...;

        foreach (ViewConfiguration viewConfiguration in ViewConfiguration.EnabledViewConfigurations)
        {
            if (viewConfiguration.IsActive && viewConfiguration.SupportedReprojectionModes.Contains(reprojectionSettings.ReprojectionMode))
            {
                viewConfiguration.SetReprojectionSettings(reprojectionSettings);
            }
        }
    }
}
```

## Depth LSR

For Depth LSR to work, the client application must supply a valid depth buffer that contains all the relevant geometry to consider during LSR.

Depth LSR attempts to stabilize the video frame based on the contents of the supplied depth buffer. As a consequence, content that hasn't been rendered to it, such as transparent objects, can't be adjusted by LSR and may show instability and reprojection artifacts.

To mitigate reprojection instability for transparent objects, you can force depth buffer writing. See the material flag *TransparencyWritesDepth* for the [Color](color-materials.md) and [PBR](pbr-materials.md) materials. Note however, that visual quality of transparent/opaque object interaction may suffer when enabling this flag.

## Planar LSR

Planar LSR does not have per-pixel depth information, as Depth LSR does. Instead it reprojects all content based on a plane that you must provide each frame.

Planar LSR reprojects those objects best that lie close to the supplied plane. The further away an object is, the more unstable it will look. While Depth LSR is better at reprojecting objects at different depths, Planar LSR may work better for content aligning well with a plane.

### Configure Planar LSR in Unity

The plane parameters are derived from a so called *focus point*. When using WMR, the focus point has to be set every frame through `UnityEngine.XR.WSA.HolographicSettings.SetFocusPointForFrame`. See the [Unity Focus Point API](/windows/mixed-reality/focus-point-in-unity) for details. For OpenXR, the focus point needs to be set via the `ReprojectionSettings` shown in the previous section.
If you don't set a focus point, a fallback will be chosen for you. However that automatic fallback often leads to suboptimal results.

You can calculate the focus point yourself, though it might make sense to base it on the one calculated by the Remote Rendering host. Call `RemoteManagerUnity.CurrentSession.GraphicsBinding.GetRemoteFocusPoint` to obtain that.

Usually both the client and the host render content that the other side isn't aware of, such as UI elements on the client. Therefore, it might make sense to combine the remote focus point with a locally calculated one.

The focus points calculated in two successive frames can be quite different. Simply using them as-is can lead to holograms appearing to be jumping around. To prevent this behavior, interpolating between the previous and current focus points is advisable.

## Next steps

* [Server-side performance queries](performance-queries.md)