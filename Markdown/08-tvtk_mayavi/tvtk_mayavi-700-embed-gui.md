

```python
%gui qt
from scpy2.tvtk import fix_mayavi_bugs, patch_pipeline_browser
fix_mayavi_bugs()
patch_pipeline_browser()
```

    WARNING:traits.has_traits:DEPRECATED: traits.has_traits.wrapped_class, 'the 'implements' class advisor has been deprecated. Use the 'provides' class decorator.


## 将TVTK和Mayavi嵌入到界面中

### TVTK场景的嵌入

> **SOURCE**

> `scpy2.tvtk.example_embed_tube`：演示如何将TVTK的场景嵌入进TraitsUI界面，可通过界面中的控件调节圆管的内径和外径。


```python
#%hide
%exec_python -m scpy2.tvtk.example_tube
```


```python
%%include python tvtk/example_tube.py 1
from traits.api import HasTraits, Instance, Range, on_trait_change
from traitsui.api import View, Item, VGroup, HGroup, Controller
from tvtk.api import tvtk
from tvtk.pyface.scene_editor import SceneEditor
from tvtk.pyface.scene import Scene
from tvtk.pyface.scene_model import SceneModel


class TVTKSceneController(Controller):
    def position(self, info):
        super(TVTKSceneController, self).position(info)
        self.model.plot() #❸


class TubeDemoApp(HasTraits):
    radius1 = Range(0, 1.0, 0.8)
    radius2 = Range(0, 1.0, 0.4)
    scene = Instance(SceneModel, ()) #❶
    view = View(
                VGroup(
                    Item(name="scene", editor=SceneEditor(scene_class=Scene)), #❷
                    HGroup("radius1", "radius2"),
                    show_labels=False),
                resizable=True, height=500, width=500)
        
    def plot(self):
        r1, r2 = min(self.radius1, self.radius2), max(self.radius1, self.radius2)
        self.cs1 = cs1 = tvtk.CylinderSource(height=1, radius=r2, resolution=32)
        self.cs2 = cs2 = tvtk.CylinderSource(height=1.1, radius=r1, resolution=32)
        triangle1 = tvtk.TriangleFilter(input_connection=cs1.output_port)
        triangle2 = tvtk.TriangleFilter(input_connection=cs2.output_port)
        bf = tvtk.BooleanOperationPolyDataFilter()
        bf.operation = "difference"
        bf.set_input_connection(0, triangle1.output_port)
        bf.set_input_connection(1, triangle2.output_port)
        m = tvtk.PolyDataMapper(input_connection=bf.output_port, scalar_visibility=False)
        a = tvtk.Actor(mapper=m)
        a.property.color = 0.5, 0.5, 0.5
        self.scene.add_actors([a])
        self.scene.background = 1, 1, 1
        self.scene.reset_zoom()
    
    @on_trait_change("radius1, radius2") #❹
    def update_radius(self):
        self.cs1.radius = max(self.radius1, self.radius2)
        self.cs2.radius = min(self.radius1, self.radius2)
        self.scene.render_window.render()        


if __name__ == "__main__":
    app = TubeDemoApp()
    app.configure_traits(handler=TVTKSceneController(app))
```

### Mayavi场景的嵌入

> **SOURCE**

> `scpy2.tvtk.example_embed_fieldviewer`：标量场观察器，演示如何将Mayavi的场景嵌入到TraitsUI的界面中


```python
#%hide
%exec_python -m scpy2.tvtk.example_embed_fieldviewer
```


```python
%%include python tvtk/example_embed_fieldviewer.py 1
import numpy as np

from traits.api import HasTraits, Float, Int, Bool, Range, Str, Button, Instance
from traitsui.api import View, HSplit, Item, VGroup, EnumEditor, RangeEditor
from tvtk.pyface.scene_editor import SceneEditor 
from mayavi.tools.mlab_scene_model import MlabSceneModel
from mayavi.core.ui.mayavi_scene import MayaviScene
from scpy2.tvtk import fix_mayavi_bugs

fix_mayavi_bugs()


class FieldViewer(HasTraits):
    
    # 三个轴的取值范围
    x0, x1 = Float(-5), Float(5)
    y0, y1 = Float(-5), Float(5)
    z0, z1 = Float(-5), Float(5)
    points = Int(50) # 分割点数
    autocontour = Bool(True) # 是否自动计算等值面
    v0, v1 = Float(0.0), Float(1.0) # 等值面的取值范围
    contour = Range("v0", "v1", 0.5) # 等值面的值
    function = Str("x*x*0.5 + y*y + z*z*2.0") # 标量场函数
    function_list = [
        "x*x*0.5 + y*y + z*z*2.0",
        "x*y*0.5 + np.sin(2*x)*y +y*z*2.0",
        "x*y*z",
        "np.sin((x*x+y*y)/z)"
    ]
    plotbutton = Button("描画")
    scene = Instance(MlabSceneModel, ()) #❶
    
    view = View(
        HSplit(
            VGroup(
                "x0","x1","y0","y1","z0","z1",
                Item('points', label="点数"),
                Item('autocontour', label="自动等值"),
                Item('plotbutton', show_label=False),
            ),
            VGroup(
                Item('scene', 
                    editor=SceneEditor(scene_class=MayaviScene), #❷
                    resizable=True,
                    height=300,
                    width=350
                ), 
                Item('function', 
                    editor=EnumEditor(name='function_list', evaluate=lambda x:x)),
                Item('contour', 
                    editor=RangeEditor(format="%1.2f",
                        low_name="v0", high_name="v1")
                ), show_labels=False
            )
        ),
        width = 500, resizable=True, title="三维标量场观察器"
    )
      
    def _plotbutton_fired(self):
        self.plot()

    def plot(self):
        # 产生三维网格
        x, y, z = np.mgrid[ #❸
            self.x0:self.x1:1j*self.points, 
            self.y0:self.y1:1j*self.points, 
            self.z0:self.z1:1j*self.points]
            
        # 根据函数计算标量场的值
        scalars = eval(self.function)  #❹
        self.scene.mlab.clf() # 清空当前场景
        
        # 绘制等值平面
        g = self.scene.mlab.contour3d(x, y, z, scalars, contours=8, transparent=True) #❺
        g.contour.auto_contours = self.autocontour
        self.scene.mlab.axes(figure=self.scene.mayavi_scene) # 添加坐标轴

        # 添加一个X-Y的切面
        s = self.scene.mlab.pipeline.scalar_cut_plane(g)
        cutpoint = (self.x0+self.x1)/2, (self.y0+self.y1)/2, (self.z0+self.z1)/2
        s.implicit_plane.normal = (0,0,1) # x cut
        s.implicit_plane.origin = cutpoint
        
        self.g = g #❻
        self.scalars = scalars
        # 计算标量场的值的范围
        self.v0 = np.min(scalars)
        self.v1 = np.max(scalars)
        
    def _contour_changed(self): #❼
        if hasattr(self, "g"):
            if not self.g.contour.auto_contours:
                self.g.contour.contours = [self.contour]
                
    def _autocontour_changed(self): #❽
        if hasattr(self, "g"):
            self.g.contour.auto_contours = self.autocontour
            if not self.autocontour:
                self._contour_changed()


if __name__ == '__main__':
    app = FieldViewer()
    app.configure_traits()
```