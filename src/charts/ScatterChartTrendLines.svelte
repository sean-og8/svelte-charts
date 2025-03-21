<svelte:options accessors={true} />

<script>	
	import { LayerCake, Svg, groupLonger} from 'layercake';
	import { scaleOrdinal, scaleLinear, scaleSymlog } from 'd3-scale';
  	import { tweened } from 'svelte/motion';
	import { cubicInOut } from 'svelte/easing';
	import { commas } from '../js/utils';

	import SetCoords from './shared/SetCoords.svelte';
	import Scatter from './shared/Scatter.svg.svelte';
	import Voronoi from './shared/Voronoi.svelte';
	import AxisX from './shared/AxisX.svelte';
	import AxisY from './shared/AxisY.svelte';
	import Legend from './shared/Legend.svelte';
	import Title from './shared/Title.svelte';
	import Footer from './shared/Footer.svelte';
	import Labels from './shared/Labels.svelte';
	import StaticLabels from './shared/StaticLabels.svelte';
	import Export from './shared/Export.svelte';
	import Table from './shared/Table.svelte';
	import MultiLine from './shared/MultiLine.svelte';

	export let data;
	export let trendData = null; // data for trend line
	export let height = 200; // number of pixels or valid css height string
	export let ssr = false;
	export let ssrWidth = 300; // for SSR only. Must be a number
	export let ssrHeight = typeof height == 'number' ? height : 200; // for SSR only. Number, or calculated from 'height'
	export let animation = true;
	export let duration = 800;
	export let xKey = 'x';
	export let yKey = null;
	export let zKey = null;
  	export let rKey = null;
	export let idKey = xKey;
	export let labelKey = idKey;
	export let xScale = 'linear';
	export let yScale = 'linear';
	export let xFormatTick = commas;
	export let yFormatTick = commas;
	export let xMax = null;
	export let xMin = null;
	export let yMax = null;
	export let yMin = null;
	export let xAxis = true;
	export let yAxis = true;
	export let xTicks = 4;
  	export let yTicks = 4;
	export let zDomain = null;
	export let textColor = '#666';
	export let tickColor = '#ccc';
	export let tickDashed = false;
	export let title = null;
	export let alt = null;
	export let footer = null;
	export let legend = false;
	export let labels = false;
	export let labelsToDisplay = [];
	export let labelContent = null;
	export let snapTicks = false;
	export let padding = { top: 0, bottom: 20, left: 35, right: 0 };
	export let buffer = 5;
	export let color = null;
	export let colors = color ? [color] : ['#206095', '#A8BD3A', '#003C57', '#27A0CC', '#118C7B', '#F66068', '#746CB1', '#22D0B6', 'lightgrey'];
	export let r = 4;
	export let interactive = true;
	export let xPrefix = "";
	export let xSuffix = "";
	export let yPrefix = "";
	export let ySuffix = "";
	export let hover = false;
	export let hovered = null;
	export let colorHover = 'orange';
	export let select = false;
	export let selected = null;
	export let colorSelect = 'black';
	export let highlighted = [];
	export let colorHighlight = 'black';
	export let overlayFill = false;
	export let output = null;
	export let mode = 'default';
	export let lineWidth = 2.5;
	export let line = true;
	export let xGridlines = true;

	let el; // Chart DOM element

	const tweenOptions = {
		duration: duration,
		easing: cubicInOut
	};
	const coords = tweened(undefined, tweenOptions);
  
  const distinct = (d, i, arr) => arr.indexOf(d) ==  i;

	function domGet(data, key, min, max) {
		let vals = data.map(d => d[key]);
		return [min ? min : vals[0] ? Math.min(...vals) : -1, max ? max : vals[0] ? Math.max(...vals) : 1];
	}
	function xDomUpdate(data, key, min, max) {
		let newDom = domGet(data, key, min, max);
		if (newDom[0] != xDom[0] || newDom[1] != xDom[1]) {
			xDomain.set(newDom);
			xDom = newDom;
		}
	}
	function yDomUpdate(data, key, min, max) {
		let newDom = key ? domGet(data, key, min, max) : yDom;
		if (newDom[0] != yDom[0] || newDom[1] != yDom[1]) {
			yDomain.set(newDom, {duration: animation ? duration : 0});
			yDom = newDom;
		}
	}
	let xDom = domGet(data, xKey, xMin, xMax);
	const xDomain = tweened(xDom, tweenOptions);
	let yDom = domGet(data, yKey, yMin, yMax);
	const yDomain = tweened(yDom, tweenOptions);


	// create long data format of grouped data
	const longData = [];
	trendData.forEach(item => {
			const existingGroup = longData.find(groupItem => groupItem[zKey] === item[zKey]);
		if (existingGroup) {
		existingGroup.values.push(item);
		} else {
		longData.push({
		[zKey]: item[zKey],
		values: [item]
		});
	}
	});
	

	$: xDomUpdate(data, xKey, xMin, xMax);
	$: yDomUpdate(data, yKey, yMin, yMax);
    $: _zDomain = Array.isArray(zDomain) ? zDomain :
		zKey && typeof zDomain === "function" ? data.map(d => d[zKey]).filter(distinct).sort(zDomain) : 
		zKey ? data.map(d => d[zKey]).filter(distinct) : null;

</script>

<div bind:this={el}>
{#if title}
  <Title>{title}</Title>
{/if}
{#if alt}
	<h5 class="visuallyhidden">{alt}</h5>
{/if}
<slot name="options"/>
<div class="chart-container" style="height: {typeof height == 'number' ? height + 'px' : height }" aria-hidden="true">
		{#if trendData}
			<LayerCake
				{padding}
				{ssr}
				height={ssr ? ssrHeight : null}
				width={ssr ? ssrWidth : null}
				x={xKey}
				y={yKey}
				z={zKey}
	            r={rKey}
				xScale={typeof xScale == 'function' ? xScale() : xScale == 'log' ? scaleSymlog() : scaleLinear()}
				yScale={typeof yScale == 'function' ? yScale() : yScale == 'log' ? scaleSymlog() : scaleLinear()}
				zScale={scaleOrdinal()}
				xDomain={$xDomain}
				yDomain={$yDomain}
				zDomain={_zDomain}
				zRange={colors}
				rRange={Array.isArray(r) ? r : [r, r]}
				data={longData}
				flatData={trendData}
				xPadding={[buffer, buffer]}
				yPadding={yKey ? [buffer, buffer] : null}
				>
				<Svg>
					{#if xAxis}
								<AxisX ticks={xTicks} formatTick={xFormatTick} {snapTicks} prefix={xPrefix} suffix={xSuffix} {textColor} {tickColor} {tickDashed} gridlines={xGridlines}/>
					{/if}
					{#if yAxis && yKey}
							<AxisY ticks={yTicks} formatTick={yFormatTick} prefix={yPrefix} suffix={ySuffix} {textColor} {tickColor} {tickDashed}/>
					{/if}
					<MultiLine/>
				</Svg>
				<LayerCake
					{padding}
					{ssr}
					height={ssr ? ssrHeight : null}
					width={ssr ? ssrWidth : null}
					x={xKey}
					y={yKey}
					z={zKey}
					r={rKey}
					xScale={typeof xScale == 'function' ? xScale() : xScale == 'log' ? scaleSymlog() : scaleLinear()}
					yScale={typeof yScale == 'function' ? yScale() : yScale == 'log' ? scaleSymlog() : scaleLinear()}
					zScale={scaleOrdinal()}
					xDomain={$xDomain}
					yDomain={$yDomain}
					zDomain={_zDomain}
					zRange={colors}
					rRange={Array.isArray(r) ? r : [r, r]}
					data={data}
					xPadding={[buffer, buffer]}
					yPadding={yKey ? [buffer, buffer] : null}
					custom={{
							type: 'scatter',
							idKey,
							labelKey,
							coords,
							colorSelect,
							colorHover,
							colorHighlight,
							padding: 1,
							animation,
							duration
						}}
					>
						<SetCoords/>
						<slot name="back"/>
						<Svg pointerEvents={interactive}>
					
								<Scatter {selected} {hovered} {highlighted} {overlayFill}/>
								{#if select || hover}
									<Voronoi {select} bind:selected {hover} bind:hovered {highlighted} on:hover on:select/>
								{/if}
								{#if labels}
									<Labels {hovered} {selected} content={labelContent}/>
									<StaticLabels content={labelContent} {labelsToDisplay}/>
								{/if}
								<slot name="svg"/>
						</Svg>
			</LayerCake>
	<slot name="front"/>
	</LayerCake>
	{/if}
</div>
<div class="visuallyhidden">
	<Table {data} key1={zKey} key2={xKey} key3={yKey} key4={rKey}/>
</div>
<slot name="legend"/>
{#if legend && _zDomain}
  <Legend domain={_zDomain} {colors} markerLength={Array.isArray(r) ? r[0] * 2 : r * 2} round={true}/>
{/if}
{#if footer}
  <Footer>{footer}</Footer>
{/if}
</div>
{#if output}
	<Export {el} {data} keys={[idKey, xKey, zKey, yKey]} {title} {output}/>
{/if}

<style>
	.chart-container {
		width: 100%;
	}
	.visuallyhidden {
		position: absolute; 
		width: 1px; 
		height: 1px; 
		margin: -1px; 
		padding: 0; 
		overflow: hidden;
		clip: rect(0,0,0,0);  
		border: 0;
	}
</style>
