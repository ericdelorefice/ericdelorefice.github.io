---
layout: page
title: Projects
permalink: /projects/
---

Coursework from CSC-1210 Web Site Development at High Point University, Fall 2026. The cards below sit in a CSS Grid that goes from one column on a phone, to two on a tablet, to three on a laptop.

<div class="project-grid">

<div class="project-card">
<h3><a href="/inspired-by/">Inspired By</a></h3>
<p>A page about a historical figure who inspires me. Built with semantic HTML and CSS Grid, responsive across three breakpoints, and layered on top of a reset stylesheet. It started as a Flexbox layout and was rebuilt twice as the class moved from Flexbox to Grid to responsive design.</p>
<span class="tag">HTML5</span><span class="tag">Grid</span><span class="tag">Responsive</span>
</div>

<div class="project-card">
<h3><a href="/js-practice/">JavaScript Practice</a></h3>
<p>In-class JavaScript. Uses getElementById to change HTML content, swap the src attribute on images, and change styles on the fly. Click the buttons to swap the light bulb and the tachometer.</p>
<span class="tag">JavaScript</span><span class="tag">DOM</span>
</div>

<div class="project-card">
<h3><a href="/bootstrap/">Bootstrap Practice</a></h3>
<p>A four column Bootstrap 4 layout built on the twelve column grid, with an alert, a set of buttons, a card and a progress bar.</p>
<span class="tag">Bootstrap 4</span><span class="tag">Components</span>
</div>

</div>

## What the class has covered

<button type="button" class="toggle-btn" onclick="document.getElementById('learned').style.display='block'">Show the list</button>
<button type="button" class="toggle-btn" onclick="document.getElementById('learned').style.display='none'">Hide</button>

<div id="learned">
<ul>
<li>Semantic HTML5 structure: header, nav, main, section, article, aside, footer</li>
<li>CSS selectors, the box model, borders, padding and margins</li>
<li>Flexbox for one dimensional layout</li>
<li>CSS Grid for two dimensional layout</li>
<li>Responsive design with media queries for mobile, tablet and desktop</li>
<li>Reset stylesheets</li>
<li>Bootstrap 4 and its twelve column grid</li>
<li>JavaScript with getElementById to change content, attributes and styles</li>
</ul>
</div>
