<script>
  import { onMount } from 'svelte';

  import * as am5 from '@amcharts/amcharts5';
  import * as am5map from '@amcharts/amcharts5/map';
  import am5locales_de_DE from '@amcharts/amcharts5/locales/de_DE';
  import am5themes_Animated from '@amcharts/amcharts5/themes/Animated';
  import am5geodata_world from '@amcharts/amcharts5-geodata/worldHigh';
  // import am5geodata_world from '@amcharts/amcharts5-geodata/worldUltra';
  import am5geodata_lang_DE from '@amcharts/amcharts5-geodata/lang/DE';

  import data from './data.json';
  import Symbols from './Symbols.svelte';

  let main;
  let index = -1;
  // index = 147;
  let item;
  let chart, backgroundSeries;
  let showGlobe = false;
  let headline = 'In 80 Jahren um die Welt';
  let year = 1943;

  onMount(() => {
    const root = am5.Root.new('chartdiv', {});

    root.locale = am5locales_de_DE;
    root.setThemes([am5themes_Animated.new(root)]);

    chart = root.container.children.push(
      am5map.MapChart.new(root, {
        panX: 'translateX',
        panY: 'translateY',
        maxPanOut: 0,
        projection: am5map.geoMercator(),
        // homeGeoPoint: { latitude: 0, longitude: 0 },
        paddingBottom: 20,
        paddingTop: 20,
        // paddingLeft: 20,
        // paddingRight: 20,
      })
    );

    ///////////////////////////////////////////////////////////////////////////
    // Create series for background fill
    // https://www.amcharts.com/docs/v5/charts/map-chart/map-polygon-series/#Background_polygon
    backgroundSeries = chart.series.push(am5map.MapPolygonSeries.new(root, {}));
    backgroundSeries.mapPolygons.template.setAll({
      fill: root.interfaceColors.get('alternativeBackground'),
      fillOpacity: 0,
      strokeOpacity: 0,
    });

    // Add background polygon
    // https://www.amcharts.com/docs/v5/charts/map-chart/map-polygon-series/#Background_polygon
    backgroundSeries.data.push({
      geometry: am5map.getGeoRectangle(90, 180, -90, -180),
    });

    ////////////////////////////////////////////////////////////////////////////

    let colors = am5.ColorSet.new(root, {});
    // colors.next();

    let polygonSeries = chart.series.push(
      am5map.MapPolygonSeries.new(root, {
        geoJSON: am5geodata_world,
        geodataNames: am5geodata_lang_DE,
        exclude: ['AQ'],
      })
    );
    polygonSeries.mapPolygons.template.setAll({
      tooltipText: '{name}',
      toggleKey: 'active',
      interactive: true,
    });

    polygonSeries.mapPolygons.template.states.create('hover', {
      fill: root.interfaceColors.get('primaryButtonHover'),
    });

    polygonSeries.mapPolygons.template.states.create('active', {
      fill: colors.getIndex(6),
    });

    polygonSeries.mapPolygons.template.states.create('mark', {
      fill: colors.getIndex(3),
    });

    let previousPolygon;

    polygonSeries.mapPolygons.template.on('active', function (active, target) {
      /*
      if (previousPolygon && previousPolygon != target) {
        // previousPolygon.set('active', false);
      }
      if (target.get('active')) {
        console.log(target);
        polygonSeries.zoomToDataItem(target.dataItem);
      } else {
        chart.goHome();
      }
      previousPolygon = target;
      */
    });

    // polygonSeries.mapPolygons.template.events.on('click', function (event) {
    //   console.log('Clicked on a thing', event.target.dataItem);
    // });

    // Set clicking on "water" to zoom out
    chart.chartContainer.get('background').events.on('click', function () {
      chart.goHome();
    });

    // destination series
    let pointSeries = chart.series.push(am5map.MapPointSeries.new(root, {}));

    pointSeries.bullets.push(function () {
      let circle = am5.Circle.new(root, {
        radius: 7,
        showTooltipOn: 'always',
        tooltipText: '{title}',
        tooltipY: 0,
        tooltip: am5.Tooltip.new(root, {}),
        fill: am5.color(0xdc3300), // colors.getIndex(15), // am5.color(0xffba00),
        stroke: root.interfaceColors.get('background'),
        strokeWidth: 2,
      });
      // circle.setAll({
      //   tooltipText: '{title}',
      //   tooltipY: 0,
      //   showTooltipOn: "always",
      //   tooltip: am5.Tooltip.new(root, {})
      // });

      return am5.Bullet.new(root, {
        sprite: circle,
      });
    });

    // Make stuff animate on load
    chart.appear(1000, 100);

    function markCountry(country, zoom) {
      const dataItem = polygonSeries.getDataItemById(country);

      if (dataItem) {
        const target = dataItem.get('mapPolygon');
        // if (target.get('active')) {
        //   target.set('active', false);
        // }
        target.states.applyAnimate('mark');
        // target.set('fill', colors.next());

        if (zoom) {
          console.log('zoom', country);
          if (showGlobe) {
            let centroid = target.geoCentroid();
            if (centroid) {
              chart.animate({ key: 'rotationX', to: -centroid.longitude, duration: 1500, easing: am5.ease.inOut(am5.ease.cubic) });
              chart.animate({ key: 'rotationY', to: -centroid.latitude, duration: 1500, easing: am5.ease.inOut(am5.ease.cubic) });
            }
          } else {
            polygonSeries.zoomToDataItem(target.dataItem);
          }
        } else {
          console.log('no zoom', country);
        }
      }
    }

    function unmarkCountry(country, active) {
      const dataItem = polygonSeries.getDataItemById(country);

      if (dataItem) {
        const target = dataItem.get('mapPolygon');
        if (active) {
          target.states.applyAnimate('active');
        }
        target.set('active', active);
      }
    }

    function showEvent(direction = 1) {
      // unmark
      if (item) {
        headline = year = item.year;
        if (Array.isArray(item.country)) {
          item.country.forEach(country => {
            unmarkCountry(country, direction > 0);
          });
        } else {
          unmarkCountry(item.country, direction > 0);
        }
      }

      index += direction;
      if (index < 0 || index >= data.length) {
        index -= direction;
        item = undefined;
        pointSeries.data.setAll([]);
        chart.goHome();
        setTimeout(() => {
          chart.set('projection', am5map.geoOrthographic());
          chart.set('panX', 'rotateX');
          chart.set('panY', 'rotateY');
          chart.series.push(
            am5map.MapPolygonSeries.new(root, {
              geoJSON: am5geodata_world,
              geodataNames: am5geodata_lang_DE,
              include: ['AQ'],
            })
          );
          backgroundSeries.mapPolygons.template.set('fillOpacity', 0.1);
          chart.animate({
            key: 'rotationX',
            from: 0,
            to: 360,
            duration: 30000,
            loops: Infinity,
          });
        }, 3000);
        return;
      }
      item = data[index];
      headline = year = item.year;

      pointSeries.data.setAll(item.cities || []);

      if (Array.isArray(item.country)) {
        item.country.forEach((country, i) => {
          markCountry(country, i === item.country.length - 1);
        });
      } else {
        markCountry(item.country, true);
      }
    }

    document.addEventListener('keydown', event => {
      switch (event.key) {
        case 'ArrowLeft':
          showEvent(-1);
          break;
        case 'ArrowRight':
          showEvent(1);
          break;
      }
    });
    window.addEventListener('showevent', () => showEvent(1));
  });

  function toggleFullscreen() {
    if (!document.fullscreenElement) {
      document.documentElement.requestFullscreen();
    } else if (document.exitFullscreen) {
      document.exitFullscreen();
    }
  }

  function toppleMap() {
    showGlobe = !showGlobe;
    if (!showGlobe) {
      chart.set('projection', am5map.geoMercator());
      chart.set('panX', 'translateX');
      chart.set('panY', 'translateY');
      chart.set('rotationY', 0);
      backgroundSeries.mapPolygons.template.set('fillOpacity', 0);
    } else {
      chart.set('projection', am5map.geoOrthographic());
      chart.set('panX', 'rotateX');
      chart.set('panY', 'rotateY');
      backgroundSeries.mapPolygons.template.set('fillOpacity', 0.1);
    }
    chart.goHome();
  }

  function goHome() {
    chart.goHome();
  }

  function zoomIn() {
    chart.zoomIn();
  }

  function zoomOut() {
    chart.zoomOut();
  }

  function setYear() {
    const newIndex = data.findIndex(i => i.year === parseInt(this.value, 10));
    console.log(this.value, year, newIndex);
    if (newIndex > 0) {
      index = newIndex - 1;
      window.dispatchEvent(new CustomEvent('showevent'));
    }
    this.blur();
    console.log(Number.isInteger(this.value), Number.isInteger(year));
  }
</script>

<Symbols />
<main bind:this={main}>
  <header>
    <input type="range" min="1943" max="2023" bind:value={year} on:change={setYear} />
    <h1>
      {index === -1 ? headline : year}
    </h1>
  </header>
  <div id="chartdiv" />

  <aside class="sidebar">
    {#if item}
      <div class="info">
        <h2>{item.date}</h2>
        {item.text}
      </div>
      {#if item.trivia}
        <div class="info">
          <h2>Trivia</h2>
          {item.trivia}
        </div>
      {/if}
    {/if}
  </aside>
</main>

<div class="control">
  <button on:click={goHome}>
    <svg>
      <use xlink:href="#icon-home" />
    </svg>
  </button>
  <button on:click={zoomIn}>
    <svg>
      <use xlink:href="#icon-zoom-in" />
    </svg>
  </button>
  <button on:click={zoomOut}>
    <svg>
      <use xlink:href="#icon-zoom-out" />
    </svg>
  </button>
  <button on:click={toppleMap}>
    <svg>
      <use xlink:href="#icon-globe" />
    </svg>
  </button>
  <button on:click={toggleFullscreen}>
    <svg>
      <use xlink:href="#icon-fullscreen" />
    </svg>
  </button>
</div>

<style>
  button {
    border: 0;
    padding: 0.5rem;
  }
  button:hover {
    background-color: #ccddee;
  }
  .control svg {
    fill: currentColor;
    width: 24px;
    height: 24px;
    display: block;
  }
  main {
    margin: auto;
    height: 100vh;
    display: grid;
    grid-template-columns: 1fr 25%;
    grid-template-rows: auto 1fr;
    grid-template-areas:
      'header header'
      'stage sidebar';
  }
  header {
    grid-area: header;
    text-align: center;
  }
  h1 {
    font-size: clamp(12px, 8vw, 128px);
    margin: 0;
  }
  h2 {
    font-size: 1.5rem;
    margin: 0 0 0.25em;
  }
  input[type='checkbox'] {
    display: none;
  }
  input[type='range'] {
    margin: 1rem 1rem 0;
    width: 98%;
  }
  input[type='number'] {
    width: 5ch;
  }
  #chartdiv {
    /*    aspect-ratio: 16/9;*/
    /*    height: 100vh;*/
    grid-area: stage;
  }
  .sidebar {
    grid-area: sidebar;
    padding: 0 1em;
  }
  .info {
    border-radius: 1em;
    box-shadow: 0 0 0.2rem #666;
    background-color: #f2f2f2;
    font-size: 1.5rem;
    margin-bottom: 1em;
    padding: 1em;
  }
  .control {
    position: fixed;
    right: 10px;
    bottom: 10px;
  }
</style>
