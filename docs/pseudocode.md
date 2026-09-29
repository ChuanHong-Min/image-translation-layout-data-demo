# 图片翻译结构化流程伪代码

## HTML 合成数据

```text
PROCEDURE render_html_synthetic_sample(html_template, layout_config):
    page <- fill_template_with_text_images_and_styles(html_template, layout_config)
    element_map <- assign_unique_color_to_each_trainable_element(page)
    rendered_page <- playwright_render(page.visible_layer)
    id_mask <- playwright_render(page.color_id_layer)

    annotations <- EMPTY_LIST
    FOR each color_id IN element_map:
        pixels <- find_pixels_with_color(id_mask, color_id)
        polygon <- trace_visible_region_boundary(pixels)
        bbox <- enclosing_box(polygon)
        annotations.append({
            element_id: element_map[color_id].id,
            type: element_map[color_id].type,
            text: element_map[color_id].text,
            bbox: bbox,
            polygon: polygon
        })

    RETURN {
        image: rendered_page,
        annotations: annotations
    }
```

## VLM 难例标注

```text
PROCEDURE extract_ocr_lines(image):
    normalized_image <- normalize_orientation_and_resolution(image)
    ocr_items <- run_text_detector_and_recognizer(normalized_image)
    lines <- EMPTY_LIST

    FOR each item IN ocr_items:
        IF item.text_is_empty:
            CONTINUE
        line <- {
            id: assign_stable_line_id(),
            text: item.text,
            polygon: item.text_region_polygon,
            bbox: enclosing_box(item.text_region_polygon),
            confidence: item.confidence
        }
        APPEND line TO lines

    RETURN sort_lines_by_top_left_position(lines)


PROCEDURE predict_line_relations(lines, image_context):
    candidate_pairs <- build_nearby_line_pairs(lines)
    relations <- EMPTY_LIST

    FOR each pair IN candidate_pairs:
        features <- collect_text_geometry_and_context_features(pair, image_context)
        label <- relation_model_predict(features)
        IF label indicates_same_semantic_group:
            APPEND {from: pair.first.id, to: pair.second.id, type: label} TO relations

    groups <- merge_lines_by_relation_graph(lines, relations)
    RETURN groups


PROCEDURE staged_vlm_annotation(image, lines):
    orientation <- ask_vlm_for_page_direction(image)
    scene_and_containers <- ask_vlm_for_scene_and_text_containers(image, lines, orientation)
    group_and_order <- ask_vlm_for_semantic_groups_and_reading_order(
        image,
        lines,
        orientation,
        scene_and_containers
    )

    RETURN {
        orientation: orientation,
        containers: scene_and_containers.containers,
        scene_type: scene_and_containers.scene_type,
        groups: group_and_order.groups,
        reading_order: group_and_order.reading_order
    }


PROCEDURE validate_schema(annotation, lines):
    errors <- EMPTY_LIST

    REQUIRE annotation.orientation EXISTS
    REQUIRE annotation.containers IS_LIST
    REQUIRE annotation.groups IS_LIST
    REQUIRE annotation.reading_order IS_LIST

    FOR each group IN annotation.groups:
        REQUIRE group.id EXISTS
        REQUIRE group.line_ids IS_LIST
        REQUIRE every line_id IN group.line_ids EXISTS IN lines

    REQUIRE reading_order_contains_known_group_ids(annotation.reading_order, annotation.groups)
    REQUIRE no_duplicate_line_assignment(annotation.groups)
    REQUIRE polygons_and_boxes_are_inside_image_bounds(annotation)

    RETURN errors


PROCEDURE repair_reading_order(annotation, lines):
    order <- annotation.reading_order
    groups <- annotation.groups

    IF order misses any group:
        APPEND missing groups using geometric_order(groups)

    IF order contains unknown group ids:
        REMOVE unknown group ids

    IF order conflicts with strong_layout_constraints:
        order <- reorder_by_columns_rows_and_container_hierarchy(groups, lines)

    annotation.reading_order <- order
    RETURN annotation


PROCEDURE export_visualization(image, lines, annotation, output_name):
    canvas <- copy_image(image)
    draw_line_regions(canvas, lines)
    draw_group_boundaries(canvas, annotation.groups)
    draw_container_regions(canvas, annotation.containers)
    draw_reading_order_indices(canvas, annotation.reading_order)
    write_review_image(canvas, output_name)


PROCEDURE build_review_artifact(image):
    lines <- extract_ocr_lines(image)
    relation_groups <- predict_line_relations(lines, image_context_from(image))
    vlm_annotation <- staged_vlm_annotation(image, lines)
    annotation <- choose_or_merge_relation_and_vlm_results(relation_groups, vlm_annotation)
    validation_errors <- validate_schema(annotation, lines)

    IF validation_errors is not empty:
        annotation <- repair_reading_order(annotation, lines)
        validation_errors <- validate_schema(annotation, lines)

    IF validation_errors is empty:
        export_visualization(image, lines, annotation, local_review_output_name())
    ELSE:
        record_case_for_manual_review(image, validation_errors)
```
