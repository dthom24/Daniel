<template>
  <div ref="slider" class="custom-zoom-slider">
    <div ref="knob" class="custom-zoom-knob"></div>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, ref } from 'vue';
import L, { DomUtil, DomEvent, Draggable, Point, Map } from 'leaflet';

export default defineComponent({
  name: 'SimpleZoomSlider',
  props: {
    map: {
      type: Object as () => Map,
      required: true
    }
  },
  setup(props) {
    const slider = ref<HTMLElement>();
    const knob = ref<HTMLElement>();

    // === Internal variables ===
    let _knobWidth = 12;
    let _k = 1;
    let _maxSteps = 1;
    let _minZoom = 0;

    /**
     * Knob class extends Leaflet Draggable
     */
    class SimpleKnob extends Draggable {
      constructor(element: HTMLElement) {
        super(element, element);

        this.on('predrag', () => {
          this._newPos.y = 0; // lock vertical movement
          this._newPos.x = adjust(this._newPos.x); // adjust horizontal
        });
      }
    }

    // Converts raw x to clamped pixel position
    function adjust(x: number): number {
      const value = toValue(x);
      const clamped = Math.max(0, Math.min(_maxSteps, value));
      return toX(clamped);
    }

    // Converts zoom step to pixel X
    function toX(value: number): number {
      return value * _k;
    }

    // Converts pixel X to zoom step
    function toValue(x: number): number {
      return x / _k;
    }

    // Update knob based on map zoom level
    function updateKnobFromZoom(map: Map) {
      if (!knob.value) return;
      const zoomValue = map.getZoom() - _minZoom;
      DomUtil.setPosition(knob.value, new Point(toX(zoomValue), 0));
    }

    onMounted(() => {
      if (!slider.value || !knob.value || !props.map) return;

      // Insert slider into leaflet zoom controls
      const zoomControlEl = document.querySelector('.leaflet-control-zoom');
      if (zoomControlEl) {
        zoomControlEl.insertBefore(slider.value, zoomControlEl.querySelector('.leaflet-control-zoom-in'));
      }

      // Now safe to calculate layout
      _knobWidth = knob.value.offsetWidth || 12;
      const sliderWidth = slider.value.offsetWidth || 100;

      _minZoom = props.map.getMinZoom();
      const maxZoom = props.map.getMaxZoom();
      _maxSteps = maxZoom - _minZoom;
      _k = (sliderWidth - _knobWidth) / _maxSteps;

      // Create draggable knob
      const simpleKnob = new SimpleKnob(knob.value);
      simpleKnob.enable();

      // When dragging, update the map zoom
      simpleKnob.on('drag', () => {
        const position = DomUtil.getPosition(knob.value!);
        const zoom = Math.round(toValue(position.x)) + _minZoom;
        props.map.setZoom(zoom);
      });

      // When zoom changes (via buttons), update the knob
      props.map.on('zoomend', () => {
        updateKnobFromZoom(props.map);
      });

      // Initialize knob position
      updateKnobFromZoom(props.map);
    });

    return {
      slider,
      knob
    };
  }
});
</script>

<style scoped>
.custom-zoom-slider {
  width: 100px;
  height: 6px;
  background: #ccc;
  position: relative;
  margin: 0 8px;
  border-radius: 3px;
  overflow: hidden;
}

.custom-zoom-knob {
  width: 12px;
  height: 12px;
  background: #333;
  border-radius: 50%;
  position: absolute;
  top: -3px;
  cursor: pointer;
  transition: transform 0.2s ease;
}
</style>
