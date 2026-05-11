# geoIACS GeoPackage template

This folder contains the [geoIACS GeoPackage template](geoIACS_template_2.0.gpkg)


# GeoPackage Database Structure (geoIACS 2.0)

This document describes the data structure of the GeoPackage (GPKG) file for the geoIACS 2.0 model. It details the architectural choices made to translate the hierarchical XML/GML schema (XSD) into a relational database model suitable for spatial analysis in standard GIS environments.

## Translating the XSD Model to GeoPackage: Key Design Choices

The transition from an XSD schema to a GeoPackage requires adapting hierarchical object-oriented concepts into a flat, relational table structure. The most significant architectural decisions revolve around the handling of spatial data.

### The Challenge of Polymorphic Geometries
In the geoIACS XSD model, features are defined with specific spatial properties:
*   **Strict Surfaces:** Elements like `ReferenceParcel`, `AgriculturalParcel`, `AgriculturalArea`, and `OtherEligibleArea` have their geometry defined as `gml:SurfacePropertyType`. These naturally translate into standard polygon tables.
*   **Flexible Geometries:** Elements like `EcoLandscapeElement`, `Site`, and `EcologicalFocusArea` have their geometry defined as `gml:GeometryPropertyType`, meaning a single conceptual feature can be represented as a Point, a LineString, or a Polygon.

While advanced spatial databases like PostGIS can natively store mixed geometry types in a single column (allowing GIS clients like QGIS to recognize them during import and dynamically load them as separate map layers), the **OGC GeoPackage standard** has different operational constraints. To guarantee full compatibility, spatial indexing, and seamless rendering across all GIS clients, a GeoPackage requires each spatial table to be registered with a single, homogeneous geometry type within its `gpkg_geometry_columns` metadata table.

### The Split-Table Strategy
To solve the polymorphic geometry challenge and fully comply with the GeoPackage standards, features with flexible geometries have been explicitly **split into separate tables** based on their spatial type (e.g., one table for points, one for lines, one for polygons). 

Depending on the complexity of the feature, two different relational strategies were applied to manage these split tables:

1.  **Normalized Structure with Views (Applied to EcoLandscapeElement):** 
    Because `EcoLandscapeElement` features share many complex attributes and relationships (like the many-to-many link with Reference Parcels), their data was normalized. A central non-spatial table (`ecolandscapeelement`) holds all alphanumeric attributes. The spatial data is isolated in three "child" tables (`ecolandscapeelementpoint`, `ecolandscapeelementlinestring`, `ecolandscapeelementpolygon`). Finally, SQL `VIEW`s were created to merge the central attributes with the specific geometries on the fly, providing ready-to-use layers for QGIS.
2.  **Flat Structure (Applied to Site and EcologicalFocusArea):**
    For features with fewer interconnected relationships, a "flat" approach was used. The attributes defined in the XSD are physically repeated across three independent spatial tables (e.g., `sitepoint`, `sitelinestring`, `sitepolygon` and `ecologicalfocusareapoint`, `ecologicalfocusarealinestring`, `ecologicalfocusareapolygon`). This provides immediate, standalone layers without the need for complex joins.

---

## Entity-Relationship (ER) Diagram

The following diagram illustrates the logical relationships between the entities in the GeoPackage, reflecting the conceptual links defined in the XSD model and listing the complete structure of all tables.

*(Note: "GEOMETRY_TABLES" blocks represent the unified schema of the separated point, linestring, and polygon tables for readability).*

```mermaid
erDiagram
    %% Core Geographic Pillar
    REFERENCEPARCEL {
        INTEGER id PK
        TEXT rpid UK
        TEXT gml_id
        Polygon geometry
        DATETIME beginlifespanversion
        DATETIME endlifespanversion
        DATE validfrom
        DATE validto
        TEXT name
        TEXT description
        TEXT identifier
    }

    AGRICULTURALPARCEL {
        INTEGER id PK
        TEXT apid UK
        TEXT gml_id
        Polygon geometry
        TEXT holdingid
        REAL declaredarea_value
        TEXT declaredarea_uom
        TEXT maincrop_href
        TEXT maincrop_title
        TEXT localisedmaincrop_href
        TEXT localisedmaincrop_title
        TEXT catchcrop_1_href
        TEXT catchcrop_1_title
        TEXT catchcrop_2_href
        TEXT catchcrop_2_title
        TEXT localisedcatchcrop_1_href
        TEXT localisedcatchcrop_1_title
        TEXT localisedcatchcrop_2_href
        TEXT localisedcatchcrop_2_title
        TEXT organicstatus_href
        TEXT organicstatus_title
        INTEGER isanc
        INTEGER isn2000
        INTEGER isrbm
        INTEGER isasd
        DATETIME beginlifespanversion
        DATETIME endlifespanversion
        DATE validfrom
        DATE validto
        TEXT relatedrpid FK
        TEXT name
        TEXT description
        TEXT identifier
    }

    AGRICULTURALAREA {
        INTEGER id PK
        TEXT aaid UK
        TEXT gml_id
        Polygon geometry
        TEXT holdingid
        REAL declaredarea_value
        TEXT declaredarea_uom
        TEXT aatype_href
        TEXT aatype_title
        INTEGER isagroforestry
        DATETIME beginlifespanversion
        DATETIME endlifespanversion
        DATE validfrom
        DATE validto
        TEXT relatedrpid FK
        TEXT name
        TEXT description
        TEXT identifier
    }

    OTHERELIGIBLEAREA {
        INTEGER id PK
        TEXT oeaid UK
        TEXT gml_id
        Polygon geometry
        TEXT holdingid
        REAL declaredarea_value
        TEXT declaredarea_uom
        TEXT oeatype_href
        TEXT oeatype_title
        INTEGER isanc
        INTEGER isn2000
        INTEGER isrbm
        INTEGER isasd
        INTEGER isformeraanowsetaside
        DATETIME beginlifespanversion
        DATETIME endlifespanversion
        DATE validfrom
        DATE validto
        TEXT relatedrpid FK
        TEXT name
        TEXT description
        TEXT identifier
    }

    %% EcoLandscapeElement (Normalized Strategy)
    ECOLANDSCAPEELEMENT {
        INTEGER id PK
        TEXT eleid UK
        TEXT gml_id
        TEXT holdingid
        REAL declaredarea_value
        TEXT declaredarea_uom
        TEXT eletype_href
        TEXT eletype_title
        TEXT eledesignation_href
        TEXT eledesignation_title
        DATETIME beginlifespanversion
        DATETIME endlifespanversion
        DATE validfrom
        DATE validto
        TEXT description
        TEXT identifier
        TEXT name
    }

    ELE_GEOMETRY_TABLES {
        INTEGER id PK
        TEXT eleid FK
        Geometry geometry
    }

    ECOLANDSCAPEELEMENT_REFERENCEPARCEL {
        INTEGER id PK
        TEXT eleid FK
        TEXT rpid FK
    }

    %% Site and Farm Animals (Flat Strategy)
    SITE_GEOMETRY_TABLES {
        INTEGER id PK
        TEXT siteid UK
        TEXT gml_id
        Geometry geometry
        TEXT holdingid
        INTEGER animalwelfare
        DATETIME beginlifespanversion
        DATETIME endlifespanversion
        DATE validfrom
        DATE validto
        TEXT description
        TEXT identifier
        TEXT name
    }

    FARMANIMALSPECIES {
        INTEGER id PK
        TEXT siteid_point FK
        TEXT siteid_linestring FK
        TEXT siteid_polygon FK
        TEXT livestock_href
        TEXT livestock_title
        TEXT aquaculture_href
        TEXT aquaculture_title
        TEXT numberofanimals_href
        TEXT numberofanimals_title
    }

    NACEACTIVITYVALUE {
        INTEGER id PK
        TEXT siteid_point FK
        TEXT siteid_linestring FK
        TEXT siteid_polygon FK
        TEXT activity_href
        TEXT activity_title
    }

    %% Ecological Focus Area (Flat Strategy)
    ECOLOGICALFOCUSAREA_GEOMETRY_TABLES {
        INTEGER id PK
        TEXT efaid UK
        TEXT gml_id
        Geometry geometry
        REAL convertedarea_value
        TEXT convertedarea_uom
        TEXT ecologicalfocusareatype_href
        TEXT ecologicalfocusareatype_title
        DATETIME beginlifespanversion
        DATETIME endlifespanversion
        DATE validfrom
        DATE validto
        TEXT description
        TEXT identifier
        TEXT name
    }

    %% Logical Relationships
    REFERENCEPARCEL ||--o{ AGRICULTURALPARCEL : "contains (relatedRP)"
    REFERENCEPARCEL ||--o{ AGRICULTURALAREA : "contains (relatedRP)"
    REFERENCEPARCEL ||--o{ OTHERELIGIBLEAREA : "contains (relatedRP)"
    
    REFERENCEPARCEL ||--o{ ECOLANDSCAPEELEMENT_REFERENCEPARCEL : "associated_via"
    ECOLANDSCAPEELEMENT ||--o{ ECOLANDSCAPEELEMENT_REFERENCEPARCEL : "associated_via"
    
    ECOLANDSCAPEELEMENT ||--o| ELE_GEOMETRY_TABLES : "has_geometry (Point/Line/Poly)"
    SITE_GEOMETRY_TABLES ||--o{ FARMANIMALSPECIES : "hosts"
    SITE_GEOMETRY_TABLES ||--o{ NACEACTIVITYVALUE : "has_activity"
