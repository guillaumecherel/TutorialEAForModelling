# Calibration, validation et sensitivity analysis of complex systems models with OpenMOLE

Guillaume Chérel, 2015-10-23



Validation: Putting a model to the test
----------------------------------------

*Associated OpenMOLE Script: [ants\_pse/ants\_pse.oms](ants_pse/ants_pse.oms)*

*Associated Article: Chérel G., Cottineau C., Reuillon R., 2015, " Beyond Corroboration: Strengthening Model Validation by Looking for Unexpected Patterns ", PLoS ONE 10(9): e0138212. doi:[10.1371/journal.pone.0138212](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0138212)*

As stated before, knowing that a model can reproduce an observed phenomenon does not ensure its validity, that is to say that we can trust it to explain the phenomenon in other experimental conditions and that its predictions are valid for other parameter values. We already established that a way to put a model to the test was to search for the different behaviors it can exhibit. The discovery of unexpected behaviors, if they disagree with the experiment or the direct observation of the system it represents, provides us with the opportunity to revise the assumptions of the model or to correct bugs in the code. It also holds for the absence of expected pattern discovery, which reveals the incapability of the model to produce such patterns. As we test a model and as we revise it, we can obtain a model we can trust more to explain and predict a phenomenon.

One can wonder, for instance, if, according to our ant colony model, the closest food source is always exploited before the furthest. We decide to search the different patterns that the model generates in terms of time to drain the closest and the furthest food sources.

As in the previous experiment, we consider a task that runs 10 replications of the model with the same given parameter values and that provides, as its output, the median pattern described in two dimensions by the variables `medFood1`, the time in which the closest food source was exhausted, and `medFood3`, the time in which the furthest food source was exhausted.

To search for diversity, we use the [PSE (Pattern Space Exploration)](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0138212) method. As all evolutionary algorithms, PSE generates new individuals by combination of parent individuals and mutation.The specificity of PSE (inspired by the [novelty search] method (http://eplex.cs.ucf.edu/noveltysearch/userspage/)) is to select the parents which patterns are rare compared to the rest of the population and to the previous generations. In order to evaluate the rarity of a pattern, PSE discretizes the pattern space, which divides this space into cells. Each time a simulation produces a pattern, a counter is incremented in the corresponding cell. PSE preferentially selects the parents whose associated cell has a low counter. By selecting the parents with rare patterns, we have a better chance to produce new individuals with behaviors never observed before.

In order to use PSE in OpenMOLE, the only thing to modify, compared to the calibration we saw in the previous section, is the evolution method. We need to provide the following parameters:
- inputs: the model parameters with their minimum and maximum bounds,
- observables: the observables measured for each simulation and for which we search for diversity,
- gridSize: the discretization step for each observable,
- reevaluate and termination have the same meaning as in the calibration example.

Here is the OpenMOLE code used for out entomological example:

    val evolution =
        BehaviourSearch (
          inputs = 
            Seq(
              diffusion -> (0.0, 99.0), 
              evaporation -> (0.0, 99.0)),
          observables = 
            Seq(
              medFood1, 
              medFood3),
          gridSize = Seq(40, 40),
          reevaluate = 0.01,
          termination = 1000000
        )

<!---
Au fur et à mesure de l'exploration, de nouveaux motifs sont découverts. La figure suivante donne le nombre de motifs connus (que l'on calcule par le nombre de cellules dont le compteur est supérieur ou égal à 1) en fonction du nombre d'évaluations.

![](ants_pse/volumeDiscovered.png) 

Lorsque ce nombre se stabilise, c'est que PSE ne fait plus de nouvelles découvertes. Il faut être prudent sur la manière d'interpréter cela. L'absence de nouvelles découvertes peut signifier que tous les motifs que peut produire le modèle ont été découverts, mais il est aussi possible que d'autres motifs existent mais que PSE n'arrive pas à les atteindre.

La figure suivante montre les motifs découverts par PSE lorsque nous avons interrompu l'exploration.

![](ants_pse/patterns.png) 

La première observation que l'on peut faire est que dans tous les motifs découverts, la source de nourriture la plus proche a été épuisée avant la source de nourriture la plus éloignée. D'autre part, il semble y avoir une borne minimum et maximum pour le temps au cours duquel la première source de nourriture est consommée.

Ces trois observations nous donne autant de point de départ de nouvelles réflexions sur le comportement collectif des fourmis. Par exemple, l'exploitation en priorité des sources de nourritures plus proches est-elle systématique? Pourrait-il exister des espèces de fourmis qui exploiterait d'abord des sources de nourriture plus éloignées que d'autres? Si l'on trouvait une telle espèce, il faudrait se demander par quel mécanisme cela est possible, et revoir le modèle pour qu'il puisse en rendre compte. Ceci illustre comment la découverte des différents comportements que peut produire le modèle peut nous amener à formuler de nouvelles hypothèses sur le système étudié, à les tester, et à réviser le modèle, en faisant avancer notre compréhension du phénomène.

Pourquoi ne pas simplement échantillonner l'espace des paramètres pour connaître les différents comportements possibles du modèle, avec des méthodes d'échantillonnage bien connues comme le LHS? Dans une expérience avec un modèle de déplacement collectif à 5 paramètres, nous avons comparé les performances de PSE et de 3 échantillonnages dans l'espace de paramètres: LHS, Sobol et une grille régulière. Les résultats représentés dans les deux figures suivantes ont montré que l'échantillonnage de l'espace de paramètres, même lorsqu'il a de bonne propriétés de couverture de l'espace comme le LHS et Sobol, peut passer à côté de nombreux motifs, et qu'il faut préférer une méthode adaptative comme PSE qui oriente la recherche en fonction des découvertes faites au cours de celle-ci. La figure suivante montre les comportements découverts par la méthode proposée (PSE pour Pattern Space Exploration), par un échantillonnage LHS et par un échantillonnage en grille régulière.

<img src="img/flockingpatternsallmethods.png" width="600" />

Chaque point représente un comportement du modèle découvert. Les comportements sont décrits en 2 dimensions: la vélocité moyenne des particules qui se déplacent et leur diffusion relative (vers 1, elles se déplacent les unes par rapport aux autres, à 0 elles restent fixes les unes par rapport aux autres et vers -1 elles se rapprochent les unes des autres).
-->

The following figure allows to compare PSE to other sampling methods in terms of efficiency.

<img src="img/flockingmethodcomparisonsimple.png" width="350" />


Translated from the french by Guillaume Chérel, *add your name here*.

[![http://creativecommons.org/licenses/by-sa/4.0/](license-by-sa.png)](http://creativecommons.org/licenses/by-sa/4.0/) Guillaume Chérel, 2015

This text is licensed under the Creative Commons Attribution-ShareAlike 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/4.0/.
