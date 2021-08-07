---
title: " "
layout: page
hero_image: /img/cactus.jpg
permalink: /research/
---

<style type="text/css">
    img { border: 1px solid #000000; }
</style>

<div class="container is-max-desktop">
    <p class="title is-2">Research</p>
</div>

<div class="container is-max-desktop">
    <hr>
	<div class="columns">
		<div class="column is-8">
			<p class="title is-3 mb-3">Pollen molecular function</p>
			To the human eye, pollen is a tiny, featureless dot of yellow, if it can be seen at all. However, hidden within this dot is a complex organism, genetically distinct and independent from its parent. It moves through the world, interacts with the environment, and competes with other pollen to complete the cycle of plant reproduction. I study these processes using molecular, genetic, and computational tools. In maize, I have identified a series of mutant genes that are associated with problems in reproduction. Experiments with these mutants have contributed to our understanding of fundamental processes in plant reproduction.
		</div>
		<div class="column is-4">
			<img src="/img/pollen_in_oil.jpg" alt="Maize pollen grains in oil">
		</div>
	</div>
	<div class="columns">
		<div class="column is-4">
			<img src="/img/thermotolerance_fig.jpg" alt="Diagram of tomato thermotolerance experiment">
		</div>
		<div class="column is-8">
			<p class="title is-3 mb-3">Plant reproduction and heat stress</p>
			Crops around the world are constantly subjected to environmental stresses that reduce their ability to produce food. Reproductive processes in plants are particularly sensitive to periods of high heat. In tomato, heat stress during pollen tube growth leads to fewer fruits and lower yield. I am currently studying the genetic basis of heat tolerance during tomato reproduction. For this project, I have collected a diverse panel of tomato varieties that show a wide range of responses to heat stress. I will observe pollen grown from these varieties across a range of temperatures using automated microscopy and computer vision. Using data from these observations, I will search for associations between variations in pollen growth and variations in the genome of each variety. These associations may highlight regions of the tomato genome that are important for heat stress responses and which can ultimately be used to breed better tomatoes.	
		</div>
	</div>
	<div class="columns">
		<div class="column is-8">
			<p class="title is-3 mb-3">Methods and data science</p>
			While DNA and RNA sequences are becoming easier to obtain, answering key scientific questions requires additional measurements of the observable traits of organisms, known as their phenotypes. High-throughput phenotyping increases the speed of these observations by automating image capture and using computer vision to measure image features. During my graduate research, I developed the Maize Ear Scanner, a high-throughput phenotyping system designed to increase the speed of maize ear phenotyping. In this system, maize ears are rotated while a video captures the entire surface of the ear. The video is then computationally projected to create a flat image of the ear’s surface. This image is processed with a deep learning object detection pipeline which identifies kernel positions and colors. >400,000 kernels have been quantified using this system, enabling the identification of several genes involved in plant reproduction.
		</div>
		<div class="column is-4">
			<img src="/img/maize_ear_scanner.jpg" alt="Maize ear scanner">
		</div>
	</div>
	<div class="columns">
		<div class="column is-4">
			<img src="/img/earvision.jpg" alt="Maize ear scanner">
		</div>
		<div class="column is-8">
			I am currently working on new phenotyping methods and new ways to understand the data that they produce. These two areas come together in my study of tomato responses to heat stress. This project will produce vast amounts of phenotypic and genomic data. In addition to creating a new high-throughput pollen phenotyping system, I am studying methods to efficiently analyze these data. These methods include statistical techniques like genome-wide association and genomic prediction, the implementation of deep learning models on high-performance computing clusters with containerized code, and the development of Shiny apps to visualize and share data with my collaborators.
		</div>
	</div>
</div>
