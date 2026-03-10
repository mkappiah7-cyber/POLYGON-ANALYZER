import geopandas as gpd
import pandas as pd
import os

# Ask user for KML file path
file_path = input("Enter the full path of the KML polygon file: ")

# Load the KML polygon layer
gdf = gpd.read_file(file_path, driver="KML")

# Ensure GPS coordinate system
gdf = gdf.to_crs(epsg=4326)

# Fix invalid geometries
gdf["geometry"] = gdf["geometry"].buffer(0)

# Create unique polygon IDs
gdf["Polygon_ID"] = range(1, len(gdf) + 1)

# Convert to projected CRS for accurate area calculation
gdf_projected = gdf.to_crs(epsg=3857)

# Calculate polygon area in hectares
gdf["Area_ha"] = gdf_projected.geometry.area / 10000

# Create new attribute fields
gdf["Overlap_IDs"] = ""
gdf["Overlap_Percent"] = ""
gdf["Bigger_Polygon"] = ""
gdf["Greater_4ha"] = gdf["Area_ha"] > 4

# List to store overlap records
overlap_records = []

# Loop through polygons to detect overlaps
for i, poly1 in gdf.iterrows():

    overlaps = []
    percents = []
    bigger_list = []

    for j, poly2 in gdf.iterrows():

        if i != j:

            if poly1.geometry.intersects(poly2.geometry):

                intersection = poly1.geometry.intersection(poly2.geometry)

                if not intersection.is_empty:

                    overlap_area = intersection.area
                    percent = (overlap_area / poly1.geometry.area) * 100

                    overlaps.append(str(poly2["Polygon_ID"]))
                    percents.append(str(round(percent, 2)))

                    # Determine which polygon is larger
                    if poly1["Area_ha"] > poly2["Area_ha"]:
                        bigger = poly1["Polygon_ID"]
                    else:
                        bigger = poly2["Polygon_ID"]

                    bigger_list.append(str(bigger))

                    overlap_records.append({
                        "Polygon_1": poly1["Polygon_ID"],
                        "Polygon_2": poly2["Polygon_ID"],
                        "Overlap_percent": round(percent, 2),
                        "Bigger_polygon": bigger
                    })

    gdf.loc[i, "Overlap_IDs"] = ",".join(overlaps)
    gdf.loc[i, "Overlap_Percent"] = ",".join(percents)
    gdf.loc[i, "Bigger_Polygon"] = ",".join(bigger_list)

# Create overlap report dataframe
overlap_df = pd.DataFrame(overlap_records)

# Safely create output file names
base = os.path.splitext(file_path)[0]

output_kml = base + "_overlap_analysis.kml"
excel_report = base + "_overlap_report.xlsx"

# Save updated KML
gdf.to_file(output_kml, driver="KML")

# Save Excel overlap report
overlap_df.to_excel(excel_report, index=False)

# Completion message
print("Processing completed successfully")
print("Updated KML saved as:", output_kml)
print("Overlap report saved as:", excel_report)
