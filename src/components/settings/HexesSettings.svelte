<script lang="ts">

    import "../../styles/settings.css";

    import ColorInputPixi from "../ColorInputPixi.svelte";
    import SelectGrid from "../SelectGrid.svelte";
    import ImageCheckbox from "../ImageCheckbox.svelte";

    import { hex_orientation } from "../../types/terrain";
    import { map_shape } from "../../types/settings";

    import { tfield } from "../../stores/tfield";
    import { resize_parameters } from "../../stores/resize_parameters";
    import { store_has_unsaved_changes } from "../../stores/flags";
    import { tl } from "../../stores/translation";

    import { get_radius_from_width_height, get_width_height_from_radius } from '../../helpers/hexHelpers'

    export let comp_coordsLayer;
    export let comp_terrainLayer;

    export let retain_positions: Function;
    export let save_old_resize_parameters: Function;
    export let renderAllHexes: Function;
    export let redrawEntireMap: Function;

    export let retainIconPosition;
    export let retainPathPosition;
    export let retainTextPosition;

    function changeOrientation() {
		let t = $tfield.hexWidth;
		$tfield.hexWidth = $tfield.hexHeight;
		$tfield.hexHeight = t;
		//$tfield.hexWidth, $tfield.hexHeight = $tfield.hexHeight, $tfield.hexWidth

		comp_terrainLayer.changeOrientation();

		$store_has_unsaved_changes = true;

		//redrawEntireMap()
	}

</script>

<div class="settings-grid">
    <label for="blankHexColor">{$tl.settings.hexes.blank_color}</label>
    <div style="display: flex; gap: 0.25em; align-items: center;">
        <ColorInputPixi
            bind:value={$tfield.blankHexColor}
            on:change={() => {
                renderAllHexes();
            }}
            id={'blankHexColor'}
        />

        <button
            style={'height: fit-content;'}
            on:click={() => {
                $tfield.blankHexColor = 0xf2f2f2;
            }}>{$tl.settings.hexes.blank_color_reset}</button
        >
    </div>

    <label>{$tl.settings.hexes.orientation}</label>
    <div style={'height: 100%; display: flex; align-items: center;'}>
        <SelectGrid
            options={[
                { title: 'Flat Top', value: hex_orientation.FLATTOP, filename: 'flatTop' },
                { title: 'Pointy Top', value: hex_orientation.POINTYTOP, filename: 'pointyTop' },
            ]}
            bind:value={$tfield.orientation}
            on:change={() => {
                changeOrientation();

                comp_coordsLayer.cullUnusedCoordinates();
                comp_coordsLayer.updateAllCoordPositions();
                comp_coordsLayer.updateAllCoordsText();
                comp_coordsLayer.populateBlankHexes();

                save_old_resize_parameters();
            }}
        />
    </div>

    {#if $tfield.mapShape == map_shape.SQUARE}
        <label>{$tfield.orientation == hex_orientation.FLATTOP ? $tl.settings.hexes.raised_column : $tl.settings.hexes.indented_row }</label>
        <span style={'height: 100%; display: flex; align-items: center;'}>
            <SelectGrid
                options={[
                    {
                        title: $tl.general.even,
                        value: 'even',
                        filename: `${$tfield.orientation == hex_orientation.FLATTOP ? 'raisedcolumn' : 'indentedrow'}even`,
                    },
                    {
                        title: $tl.general.odd,
                        value: 'odd',
                        filename: `${$tfield.orientation == hex_orientation.FLATTOP ? 'raisedcolumn' : 'indentedrow'}odd`,
                    },
                ]}
                bind:value={$tfield.raised}
                on:change={() => {
                    if ($tfield.orientation == hex_orientation.FLATTOP) {
                        comp_terrainLayer.square_updateRaisedColumn();
                    } else {
                        comp_terrainLayer.square_changeIndentedRow();
                    }
                    comp_coordsLayer.cullUnusedCoordinates();
                }}
            />
        </span>
    {/if}

    <label for="hexWidth">{$tl.settings.hexes.width}</label>
    <input
        id="hexWidth"
        type="number"
        min={1}
        bind:value={$tfield.hexWidth}
        on:change={(e) => {
            if (Number.isNaN(e.target.valueAsNumber)) {
                $tfield.hexWidth = $resize_parameters.old_hex_width;
                return;
            }

            redrawEntireMap();
            comp_coordsLayer.updateAllCoordPositions();
            retain_positions();
            save_old_resize_parameters();
            
        }}
    />

    <label for="hexHeight">{$tl.settings.hexes.height}</label>
    <input
        id="hexHeight"
        type="number"
        min={1}
        bind:value={$tfield.hexHeight}
        on:change={(e) => {
            if (Number.isNaN(e.target.valueAsNumber)) {
                $tfield.hexHeight = $resize_parameters.old_hex_height;
                return;
            }
            redrawEntireMap();
            comp_coordsLayer.updateAllCoordPositions();
            retain_positions();
            save_old_resize_parameters();
        }}
    />

    <label for="hex-radius">{$tl.settings.hexes.size_by_radius}</label>
    <span>
        <button on:click={() => {

                let currentRadius = get_radius_from_width_height($tfield.hexWidth, $tfield.hexHeight, $tfield.orientation)

                let radius = +prompt("Enter hex radius", currentRadius.toString());
                console.log(radius);
                if (Number.isNaN(radius)|| radius < 1) return;	
                
                let new_dims = get_width_height_from_radius(radius, $tfield.orientation)
                
                $tfield.hexWidth = new_dims.width
                $tfield.hexHeight = new_dims.height
                
                redrawEntireMap();
                
                retain_positions();
                save_old_resize_parameters();

                comp_coordsLayer.updateAllCoordPositions();
            }}>Set</button>
    </span>

    <!--
    <label for="mapShape">Map Type</label>
    <select id="mapShape" bind:value={$tfield.mapShape}>
        <option value={map_shape.SQUARE}>Square</option>
        <option value={map_shape.RADIAL}>Radial</option>
    </select>
    -->

    <label title={$tl.settings.hexes.retain_position_explanation}>{$tl.settings.hexes.retain_position}<sup id="retain-position-tip" title={$tl.settings.hexes.retain_position_explanation}>?</sup></label>
    <div id="retain-position-container">
        <div id="retain-position-grid">
            <ImageCheckbox image_filename={ "/assets/img/tools/icon.svg" } title={$tl.settings.hexes.retain_icons} bind:checked={ retainIconPosition } />
            <ImageCheckbox image_filename={ "/assets/img/tools/path.svg" } title={$tl.settings.hexes.retain_paths} bind:checked={ retainPathPosition } />
            <ImageCheckbox image_filename={ "/assets/img/tools/text.svg" } title={$tl.settings.hexes.retain_text} bind:checked={ retainTextPosition } />
        </div>
    </div>
</div>

<style>

    #retain-position-container {
            width: 100%;
            height: 100%;
            display: flex;
            align-items: center;
            justify-content: center;
    }

    #retain-position-grid {
            height: 2em;
            display: flex;
            border-radius: var(--small-radius);
            overflow: hidden;
    }

    #retain-position-tip {
        color: var(--hexfriend-green);
    }

</style>
