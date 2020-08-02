# Guwahati-3D-map-Street-Map-Overlay
Tutorial on generating 3D map with street map image overlay using R
![Demo1](https://github.com/Hwoabam/Guwahati-3D-map-Street-Map-Overlay/blob/master/Media/Animation/OSM.gif)

The packages required:
```{r}
library(rayshader)
library(magick)
library(sp)
library(raster)
library(rgdal)
library(rgeos)
library(leaflet)
```
A bounding box is defined after assigning the coordinates to the top left and bottom right corners. Also using a transparent fill along with the boundary. Using the leaflet package, the location of the box with respect to the map is affirmed.  
```{r fig1, fig.height = 15, fig.width = 10, align= "center"}
# define bounding box with longitude/latitude coordinates
bbox <- list(
  p1 = list(long = 91.8700, lat = 26.3056),
  p2 = list(long =91.5292, lat =26.0660)
)

leaflet() %>%
  addTiles() %>% 
  addRectangles(
    lng1 = bbox$p1$long, lat1 = bbox$p1$lat,
    lng2 = bbox$p2$long, lat2 = bbox$p2$lat,
    fillColor = "transparent"
  ) %>%
  fitBounds(
    lng1 = bbox$p1$long, lat1 = bbox$p1$lat,
    lng2 = bbox$p2$long, lat2 = bbox$p2$lat,
  )
```
![Leaflet](https://github.com/Hwoabam/Guwahati-3D-map-Street-Map-Overlay/blob/master/Media/Snapshots/Capture23.PNG)

SRTM elevation file is used for this plot and downloaded from Derek Watkin's -"30 meter SRTM tile downloader". The downloaded SRTM hgt data is converted to a matrix. Then a heat map for elevation is produced.
```{r fig2, fig.height = 15, fig.width = 10, align= "center"}
Elevation_File <- raster("D:/Assam maps/New Folder/N26E091.hgt")
Elevation_File
#reproject the DEM
projectRaster(Elevation_File, 
              crs= "+proj=longlat +datum=WGS84 +no_defs +ellps=WGS84 +towgs84=0,0,0")
#plot raster DEM
height_shade(raster_to_matrix(Elevation_File)) %>%
  plot_map()
```
![Elevation heatmap](https://github.com/Hwoabam/Guwahati-3D-map-Street-Map-Overlay/blob/master/Media/Plots/Elevation_heatmap.png)

The crop function is used to crop the require area from the elevation matrix by using the extents defined from the bounding box created previously. 
```{r}
top_lefty=bbox$p1$long
top_leftx=bbox$p1$lat
bot_righty= bbox$p2$long
bot_rightx=bbox$p2$lat
distl2l= top_lefty-bot_righty  
scl=1-(20/(distl2l*111))
top_left = c(top_lefty, top_leftx)
bot_right   = c(bot_righty, bot_rightx)
extent_latlong = SpatialPoints(rbind(top_left, bot_right), proj4string=sp::CRS("+proj=longlat +ellps=WGS84 +datum=WGS84"))
e = extent(extent_latlong)
e
scl
elevation_cropped = crop(Elevation_File, e)
el_matrix = raster_to_matrix(elevation_cropped)
```
Ambient shade and ray shade functions are used at suitable scales for shadow detailing and watermap matrix is defined for the water rendering purposes:
```{r fig3, fig.height = 15, fig.width = 10, align= "center"}
# calculate rayshader layers
ambmat <- ambient_shade(el_matrix, zscale = 30)
raymat <- ray_shade(el_matrix, zscale = 30, lambert = TRUE)
watermap <- detect_water(el_matrix)
# plot 2D
el_matrix %>%
  sphere_shade(texture = "imhof4") %>%
  add_water(watermap, color = "imhof4") %>%
  add_shadow(raymat, max_darken = 0.5) %>%
  add_shadow(ambmat, max_darken = 0.5) %>%
  plot_map()
```
![Default texture plot](https://github.com/Hwoabam/Guwahati-3D-map-Street-Map-Overlay/blob/master/Media/Plots/imhof4.png)

The overlay image is downloaded from openstreetmaps.org in PNG format and is called and read using readPNG():  
```{r}
overlay_file <- "C:/Users/asus/Downloads/map (1).png"
overlay_img <- png::readPNG(overlay_file)
```
![Final 2D plot](https://github.com/Hwoabam/Guwahati-3D-map-Street-Map-Overlay/blob/master/Media/Plots/map(1).png)

A overlay image is then transposed upon the previously configured elevation plot with an opacity of 75% of the original image:
```{r fig4, fig.height = 15, fig.width = 10, align= "center"}
el_matrix %>%
  sphere_shade(texture = "imhof4") %>%
  add_shadow(raymat, max_darken = 0.5) %>%
  add_shadow(ambmat, max_darken = 0.5) %>%
  add_overlay(overlay_img, alphalayer = 0.75) %>%
  plot_map()
```
![Final 2D plot](https://github.com/Hwoabam/Guwahati-3D-map-Street-Map-Overlay/blob/master/Media/Plots/OSmap.png)

Using the Plot generated previoulsy and the elevation matrix, providing a zscale=7.5 and viewpoint parameters as desired: 
```{r fig5, fig.height = 15, fig.width = 10, align= "center"}
rgl::clear3d()
el_matrix %>% 
  sphere_shade(texture = "imhof4") %>% 
  add_overlay(overlay_img, alphalayer = 0.75) %>%
  add_shadow(raymat, max_darken = 0.5) %>%
  add_shadow(ambmat, max_darken = 0.5) %>%
  plot_3d(el_matrix, zscale = 7.5, windowsize = c(1200, 1000),
          water = TRUE, soliddepth = -max(el_matrix)/zscale, wateralpha = 0,
          shadowdepth = -150,
           zoom=0.5, phi=45,theta=-45,fov=70, shadowcolor = "#523E2B")
render_scalebar(limits=c(20,10,0),label_unit = "km",position = "S", y=50,scale_length = c(scl,1))
render_compass(position = "W" )
render_snapshot(title_text = "Guwahati Open street Map | DEM: 30m SRTM",title_bar_color = "black", title_color = "white", title_bar_alpha = 1)
```
![3D plot](https://github.com/Hwoabam/Guwahati-3D-map-Street-Map-Overlay/blob/master/Media/Snapshots/snap.png)

Conversion of the map to a video footage of its planar rotation using snapshots taken at 1440 intervals during the rotation at 60 fps with the help of ffmpeg
```{r}
angles= seq(0,360,length.out = 1441)[-1]
for(i in 1:1440) {
render_camera(theta=-45+angles[i])
  render_snapshot(filename = sprintf("guwahati_osm%i.png", i), 
                  title_text = "Guwahati Open street Map | DEM: 30m SRTM",
                  title_bar_color = "black", title_color = "white", title_bar_alpha = 1)
}
rgl::rgl.close()
system("ffmpeg -framerate 60 -i guwahati_osm%d.png -vcodec libx264 -an Guwahati_Street_Map.mp4 ")
```


