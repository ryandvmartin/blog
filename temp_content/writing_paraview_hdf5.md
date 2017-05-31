+++
tags = ['Paraview', 'HDF5', 'Big Data']
date = "2017-02-11T20:10:39-07:00"
title = "Generating XDMF files to read HDF5 in Paraview"
summary = """
Writing out XMDF files to read HDF5 binary formatted data files in Paraview
"""
highlight = true
math = true
image = ""
+++

In geostatistics, we are often required to view large 3D models with $M$ variables simulated on the $[nx, ny, nz]$ dimensioned grid. The number of cells in the model can quickly approach the 10's of millions, depending on the project and the purpose of the model. Binary storage is increasingly important for such models for practical reasons since: 1) we must store the models; and 2) we need a way to quickly work with the models including numerical processing and visualizations.

Enter [HDF5](https://support.hdfgroup.org/HDF5/), the open source binary format that has been adopted by many in the data science and scientific communities. Most scientific computing languages and software in some form or another support this binary storage format. HDF5 is available as compiled libraries for Fortran projects, C++ projects, has implementations for Python, Matlab and readers of HDF5 files are available in Paraview to speed up visualization of large datasets.

Paraview uses the [XDMF format](http://www.xdmf.org/index.php/XDMF_Model_and_Format) to configure HDF5 file reads. The XDMF format provides a convenient method to define the geometry format, properties of the visualization, and the path to the relevant data contained in the corresponding HDF5 file. Many examples are shown [on the XDMF](http://www.xdmf.org/index.php/XDMF_Model_and_Format) example site.

An example of an HDF5 format used to export a

```html
<?xml version="1.0" ?>
    <!DOCTYPE Xdmf SYSTEM "Xdmf.dtd" []>
    <Xdmf Version="2.0">
     <Domain>
       <Grid Name="mesh" GridType="Uniform">
         <Topology TopologyType="3DCoRectMesh" Dimensions="55 70 95"/>
          <Geometry Type="ORIGIN_DXDYDZ">
            <DataItem Format="XML" Dimensions="3">156.6 897.2 4.9</DataItem>
            <DataItem Format="XML" Dimensions="3">13.9  13.9  13.8</DataItem>
          </Geometry>
           <Attribute Name="cat1"
             AttributeType="Scalar"
             Center="Node">
           <DataItem Dimensions="95 70 55" NumberType="Float"
                     Precision="8" Format="HDF">
            h5datafile.hvtk:data/cat1/data
           </DataItem>
         </Attribute>
           <Attribute Name="cat2"
             AttributeType="Scalar"
             Center="Node">
           <DataItem Dimensions="95 70 55" NumberType="Float"
                     Precision="8" Format="HDF">
            h5datafile.hvtk:data/cat2/data
           </DataItem>
         </Attribute>
           <Attribute Name="cat3"
             AttributeType="Scalar"
             Center="Node">
           <DataItem Dimensions="95 70 55" NumberType="Float"
                     Precision="8" Format="HDF">
            h5datafile.hvtk:data/cat3/data
           </DataItem>
         </Attribute>
           <Attribute Name="cat4"
             AttributeType="Scalar"
             Center="Node">
           <DataItem Dimensions="95 70 55" NumberType="Float"
                     Precision="8" Format="HDF">
            h5datafile.hvtk:data/cat4/data
           </DataItem>
         </Attribute>
           <Attribute Name="cat5"
             AttributeType="Scalar"
             Center="Node">
           <DataItem Dimensions="95 70 55" NumberType="Float"
                     Precision="8" Format="HDF">
            h5datafile.hvtk:data/cat5/data
           </DataItem>
         </Attribute>
           <Attribute Name="cat6"
             AttributeType="Scalar"
             Center="Node">
           <DataItem Dimensions="95 70 55" NumberType="Float"
                     Precision="8" Format="HDF">
            h5datafile.hvtk:data/cat6/data
           </DataItem>
         </Attribute>

      </Grid>
     </Domain>
    </Xdmf>

```
