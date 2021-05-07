---
title: " "
layout: page
hero_image: /img/claytonia.jpg
permalink: /publications/
---
<h1>Publications</h1>
<div class="container-fluid">

        <hr>
        <div class="row" style="padding-top: 60px; margin-top: -60px;" id="publication pubmed ID">
        <div style="font-size: 120% !important; width: 100%">Publication title goes here</div><div>The authors go here</div>
        </div>
        <div class="row" style="padding-top: 20px; margin-top: - 20px;">
        	<div class="col-sm-6">
        		<img class = "img-fluid" src = "/img/warman_2020_plos_genetics.jpg" alt = "Plot of transmission rates of mutant alleles" style="max-height: 200px;">
        	</div>
        	<ul class="col-sm-6">
        			<strong>Access the paper</strong>

                                <li>DOI: <a href="https://doi.org/10.1371/journal.pgen.1008462" alt = "DOI link: 10.1371/journal.pgen.1008462"> 10.1371/journal.pgen.1008462
</a></li>
        			<li>PMID: <a href="https://pubmed.ncbi.nlm.nih.gov/32236090/" alt = "pubmed link: 32236090"> 32236090</a></li>
                                <li>Biorxiv Preprint: <a href="http://dx.doi.org/10.1101/{{publication.biorxiv}}" alt = "biorxiv preprint link: {{publication.biorxiv}}">

        			<!-- PDF -->
        			<li><a href="https://cedarwarman.github.io/pdf/warman_2020_plos_genetics.pdf" alt = "PDF"> Full Text PDF</a></li>

        			<!-- Datasets - optional -->
        			{% if publication.data %}
        			<li>Online Dataset{% if publication.data.size > 1 %}s{% endif %}:
        				{% if publication.data.size > 1 %}
        				<ul>
        					{% for dataset in publication.data %}
        					<li><a href="http://dx.doi.org/{{dataset}}" alt = "sbgrid data repository">doi:{{dataset}}</a></li>
        					{% endfor %}
        				</ul>
        				{% else %}
        				<a href="http://dx.doi.org/{{publication.data}}" alt = "sbgrid data repository">doi:{{publication.data}}</a>
        				{% endif %}
        			</li>
        			{% endif %}

        			<!--additional links - optional-->
                                {% if publication.links %}
                                <strong>Additional Link{% if publication.links.size > 1 %}s{% endif %}</strong>:
                                        {% for link in publication.links %}
                                        <li><a href="{{link.url}}" alt="{{link.name}}">{{link.name}}</a></li>
                                        {% endfor %}
                                {% endif %}
        	</ul>
        </div>
        <br>
</div>

