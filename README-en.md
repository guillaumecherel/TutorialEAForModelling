# Calibration, validation et sensitivity analysis of complex systems models with OpenMOLE

Guillaume Chérel, 2015-10-23

Translated from french by Guillaume Chérel, Mathieu Leclaire, Juste Raimbault, Julien Perret.

[![http://creativecommons.org/licenses/by-sa/4.0/](license-by-sa.png)](http://creativecommons.org/licenses/by-sa/4.0/) Guillaume Chérel, 2015

This text is licensed under the Creative Commons Attribution-ShareAlike 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/4.0/.



Calibrer un modèle pour reproduire des motifs attendus
------------------------------------------------------

*Script OpenMOLE associé: [ants\_calibrate/ants\_calibrate.oms](ants_calibrate/ants_calibrate.oms)*

*Article associé: Schmitt C, Rey-Coyrehourcq S, Reuillon R, Pumain D, 2015, "Half a billion simulations: evolutionary algorithms and distributed computing for calibrating the SimpopLocal geographical model" Environment and Planning B: Planning and Design, 42(2), 300-315. <https://hal.archives-ouvertes.fr/hal-01118918/document>*

Voyons comment OpenMOLE peut nous aider à rechercher des valeurs de paramètres avec lesquels un modèle reproduit un motif que l'on cherche à expliquer.

Reprenons l'exemple des fourmis. Imaginons que l'on ait fait une expérience où l'on a placé trois sources de nourriture autour d'une fourmilière, et que l'on ait mesuré le temps passé pour que chaque source soit entièrement épuisée. On a mesuré que la première source a été épuisée en 250 secondes, la deuxième en 400 secondes et la troisième en 800 secondes. Si notre modèle est juste, il devrait pouvoir reproduire ces mesures. Peut-on trouver des valeurs de paramètres pour lesquels il les reproduit?

On peut traduire cette question en un problème d'optimisation. Il s'agit de rechercher des valeurs de paramètres qui minimisent la différence entre les temps mesurés dans l'expérience et les temps mesurés en simulation, donné par l'expression:

    |250 - simuFood1| + |400 - simuFood2| + |800 - simuFood3|

Pour répondre à cette question avec OpenMOLE, il faut écrire un workflow qui décrit: 1. comment simuler le modèle et calculer la distance entre la simulation et les mesures issues de l'expérience, 2. comment minimiser cette distance, 3. comment distribuer les calculs en parallèle.

Le premier point relève de notions de base d'OpenMOLE dans lesquels nous ne rentrerons pas ici en détails. Admettons simplement que nous avons une tâche replicateModel qui exécute 10 réplications du modèle avec des valeurs de paramètres données, et qui calcule la distance médiane entre les résultats des simulations et des mesures expérimentales (basée sur l'expression ci-dessus), et associe au prototype foodTimesDifference.

Pour répondre au second point, OpenMOLE nous permet d'utiliser l'algorithme NSGA2, qui est un algorithme génétique d'optimisation multi-critères. Dans OpenMOLE, NSGA2 prend les paramètres suivants: - mu: un nombre d'individus à générer aléatoirement pour initialiser la population, - inputs: une séquence de paramètres du modèle pour lesquels on cherche des valeurs, et leurs bornes minimum et maximum, - objectives: une séquence de variables à minimiser, - reevaluate: une probabilité, lorsqu'on génère un nouvel individu, d'en prendre un tel quel dans la population précédente pour qu'il soit réévalué, - et un critère de terminaison.

Voici le code OpenMOLE associé:

    val evolution =
      NSGA2(
        mu = 200,
        inputs = Seq(diffusion -> (0.0, 99.0), evaporation -> (0.0, 99.0)),
        objectives = Seq(foodTimesDifference), //ici, il n'y a qu'un objectif
        reevaluate = 0.01,
        termination = 1000000
      )

La variable `foodTimesDifference` est un prototype du workflow d'OpenMOLE qui représente la somme des différences absolues entre les temps mesurés par expérience et les temps mesurés en simulation, comme dans l'expression ci-dessus. Comme le modèle est stochastique, cette valeur est définie dans le workflow comme la médiane de plusieurs réplications du modèle avec les mêmes valeurs de paramètres. L'algorithme NSGA2 va chercher à minimiser cette valeur.

Le paramètre `reevaluate` est utile lorsque le modèle est stochastique. Par chance, il se peut qu'une simulation ou un ensemble de réplications donne un résultat très bon mais peu reproductible. On préfère garder les individus qui donnent de bons résultats en moyenne. Lorsqu'un individu est très bon, il a une plus grande chance d'être sélectionné pour être réévalué. Si sa performance était un coup de chance, il est probable qu'une nouvelle évaluation donne un moins bonne performance, et donc que l'individu soit abandonné au profit d'autres individus plus robustes.

Enfin, il faut répondre au troisième point et décrire comment les calculs sont distribués. Il y a plusieurs approches possibles dans OpenMOLE: générationnelle, steady state et en steady state en îlots.

La première consiste à générer λ individus à chaque génération et à tous les évaluer en distribuant leurs évaluations sur les différentes unités de calcul disponibles. Pour continuer l'étape suivante, il faut attendre que tous les individus aient été évalués. Si certains prennent plus de temps que d'autres, on peut se retrouver dans des cas où on doit attendre que les plus long se terminent avant de continuer, alors que la plupart des unités de calcul sont inoccupées. C'est une perte de temps de calcul.

La seconde approche consiste à commencer avec μ individus et à en lancer au maximum autant qu'il y a d'unités de calcul disponible. Dès qu'une évaluation se termine, on l'intègre à la population et on en génère un nouveau que l'on relance immédiatement sur l'unité de calcul qui vient de se libérer. Cette méthode utilise continuellement les unités de calculs. C'est l'approche recommandée pour lancer une évolution sur un cluster.

La troisième approche, island steady state, convient particulièrement au calcul sur grille où l'accès aux noeuds de calculs à un coût important (par exemple à cause du temps d'attente pour qu'un noeud se libère). Au lieu de lancer seulement l'évaluation des individus sur les unités de calcul distribuées, elle consiste à lancer des algorithmes évolutifs entiers pour une période de temps fixé (par exemple, 1h). Lorsqu'une évolution se termine, sa population finale est intégrée à la population globale, puis une nouvelle population est générée et sert de population de départ à une nouvelle évolution distribuée.

Pour notre exemple, voyons comment utiliser l'approche steady state simple:

    val (puzzle, ga) = SteadyGA(evolution)(replicateModel, 40)

On passe à `SteadyGA` la méthode d'évolution que l'on a décrite plus haut, et la tâche à exécuter. Le dernier paramètre correspond au nombre d'évaluations qui sont exécutées en parallèle. SteadyGA lance de nouvelles évaluations tant que le nombre d'évaluations en cours d'exécution est inférieur à cet entier.

`SteadyGA` renvoi deux variables que l'on a appelé dans cet exemple `puzzle` et `ga`. Le second contient les informations sur l'évolution en cours. Elle permet de définir des hooks pour enregistrer la population en cours dans des fichiers csv ou d'afficher la génération en cours. La ligne suivante enregistre la population correspondant à chaque génération dans un fichier `results/population#.csv`, ou `#` est replacé par le numéro de la génération:

    val savePopulationHook = SavePopulationHook(ga, workDirectory / "results")

La ligne suivante affiche dans la console le numéro de la génération:

    val display = DisplayHook("Generation ${" + ga.generation.name + "}")

Dans OpenMOLE, un puzzle est un ensemble de tâches et de transitions qui décrivent un morceau de workflow. La variable `puzzle` contient le puzzle OpenMOLE qui déroule l'évolution. On utilise cette variable pour construire le puzzle final qui va être exécuté, et qui contient les hooks que l'on vient de définir:

    (puzzle hook savePopulationHook hook display)

Lorsqu'on lance le workflow OpenMOLE, l'évolution va progressivement produire les valeurs de paramètres avec lesquelles le modèle reproduit les mesures expérimentales. Voici l'évolution de la distance entre les simulations et les mesures expérimentales au fil des évaluations successives.

![](ants_calibrate/fitnessVSEval.png) 

Lorsque l'évolution se stabilise, on peut conclure que l'on a trouvé ou non des valeurs de paramètres avec lesquelles le modèle reproduit les données, et si oui, on peut conclure que le modèle est une explication possible du phénomène observé.

|  diffusion|  evaporation|  foodDifference|
|----------:|------------:|---------------:|
|      99.00|         5.37|           53.00|
|      45.19|         8.12|           40.50|
|      27.84|         8.77|           36.00|
|      66.19|         6.74|           56.50|
|      99.00|         5.49|           55.50|
|      64.42|         5.60|           57.50|
|      71.17|         5.61|           15.50|
|      68.10|         5.18|           49.00|
|      78.39|         5.59|           37.00|
|      78.39|         5.57|           57.00|
|      59.09|         3.72|           49.00|
|      51.71|         7.23|           58.50|
|      66.60|         5.26|           52.50|
|      21.36|         8.87|           65.50|
|      64.42|         5.60|           53.50|
|      92.45|         5.30|           58.50|
|      47.85|         6.98|           59.00|
|      68.10|         5.42|           58.50|
|      44.72|         7.09|           60.00|
|      79.39|         5.60|           59.50|


Validation: Putting a model to the test
----------------------------------------

*Associated OpenMOLE Script: [ants\_pse/ants\_pse.oms](ants_pse/ants_pse.oms)*

*Associated Article: Chérel G., Cottineau C., Reuillon R., 2015, " Beyond Corroboration: Strengthening Model Validation by Looking for Unexpected Patterns ", PLoS ONE 10(9): e0138212. doi:[10.1371/journal.pone.0138212](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0138212)*

As stated before, knowing that a model can reproduce an observed phenomenon does not ensure its validity, that is to say that we can trust it to explain the phenomenon in other experimental conditions and that its predictions are valid for other parameter values. We already established that a way to put a model to the test was to search for the different behaviors it can exhibit. The discovery of unexpected behaviors, if they disagree with the experiment or the direct observation of the system it represents, provides us with the opportunity to revise the assumptions of the model or to correct bugs in the code. It also holds for the absence of expected pattern discovery, which reveals the incapability of the model to produce such patterns. As we test a model and as we revise it, we can obtain a model we can trust more to explain and predict a phenomenon.

One can wonder, for instance, if, according to our ant colony model, the closest food source is always exploited before the furthest. We decide to search the different patterns that the model generates in terms of time to drain the closest and the furthest food sources.

As in the previous experiment, we consider a task that runs 10 replications of the model with the same given parameter values and that provides, as its output, the median pattern described in two dimensions by the variables `medFood1`, the time in which the closest food source was exhausted, and `medFood3`, the time in which the furthest food source was exhausted.

To search for diversity, we use the [PSE (Pattern Space Exploration)](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0138212) method. As all evolutionary algorithms, PSE generates new individuals by combination of parent individuals and mutation.The specificity of PSE (inspired by the [novelty search method](http://eplex.cs.ucf.edu/noveltysearch/userspage/)) is to select the parents which patterns are rare compared to the rest of the population and to the previous generations. In order to evaluate the rarity of a pattern, PSE discretizes the pattern space, which divides this space into cells. Each time a simulation produces a pattern, a counter is incremented in the corresponding cell. PSE preferentially selects the parents whose associated cell has a low counter. By selecting the parents with rare patterns, we have a better chance to produce new individuals with behaviors never observed before.

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

As the exploration progresses new patterns are discovered. The following figure gives the number of known patterns (the number of cells with a counter value greater than 0) with respect to the number of evaluations.

![](ants_pse/volumeDiscovered.png) 

When this number stabilizes, PSE does not make new discoveries anymore. One has to be careful when interpreting this. Indeed, the absence of new discoveries can mean that all the patterns that the model can produce have been discovered, but it is also possible that other patterns exist but that PSE could not reach them.

The following figure shows the patterns discovered by PSE when we interrupted the exploration.

![](ants_pse/patterns.png) 

The first observation that can be made is that all patterns have indeed been discovered: the closest food source has been drained before the furthest one. Besides, there seems to be minimum and maximum bounds for the time during which the first food source is consumed.

These three observation give us as many starting points for further reflections on the collective behavior of ants. For instance, is the exploration of the closest food sources first systematic? Could there be ant species that would explore further food sources than others first? If we found such a species, we would have to wonder which mechanisms make it possible and revise the model to take them into account. This illustrates how the discovery of the different behaviors the model is able to produce can lead us to formulate new hypotheses of the system under study, to test them and to revise the model, thus enhancing our understanding of the phenomenon.

Why not simply sample the parameter space in order to know the different potential behaviors of the model using well known sampling methods such as LHS? In the context of an experiment using a collective motion model with 5 parameters, we compared the performances of PSE and 3 samplings in the parameter space: LHS, Sobol and a regular grid. The results presented in the next two figures show that the sampling of the parameter space, even with good coverage properties such as LHS and Sobol, can miss several patterns. Adaptative methods, such as PSE, that orient the search according to the discoveries made along the way, are preferable. The following figure shows the behaviors discovered by the proposed method (PSE for Pattern Space Exploration), by a LHS sampling and a regular grid.

<img src="img/flockingpatternsallmethods.png" width="600" />

Each point represents a discovered behavior of the model. The behaviors are described in two dimensions: the average velocity of the particles and their relative diffusion (towards 1, they move away from each other, at 0, they do not move relatively to each other, towards -1, they get closer to each other).

The following figure allows to compare PSE to other sampling methods in terms of efficiency.

<img src="img/flockingmethodcomparisonsimple.png" width="350" />

Sensitivity analysis: Profiles
--------------------------------

*Linked OpenMOLE script: [ants\_profiles/ants\_profiles.oms](ants_profiles/ants_profiles.oms)*

*Article: Reuillon R., Schmitt C., De Aldama R., Mouret J.-B., 2015, "A New Method to Evaluate Simulation Models: The Calibration Profile (CP) Algorithm", JASSS : Journal of Artificial Societies and Social Simulation, Vol. 18, Issue 1, <http://jasss.soc.surrey.ac.uk/18/1/12.html>*

The method we now present aims at understanding better how the model works in focusing on the impact of the different parameters of the model. In our Anthills example, we previously calibrated the model to enforce it to reproduce fake experimental measurements. We would like to know whether the model can reproduce this pattern for other parameter values. It is possible for instance, that a parameter is crucial and yet the model cannot reproduce the experimental measurements for a different value other than the one found with the calibration. It is also possible, on the contrary, that another parameter is not essential at all, that is, the model can reproduce the experimental measurements whatever its value. To establish the relevancy of our model parameter, we will set the parameters profiles for the model and for the targeted pattern, as follows:

We first establish the profile of the evaporation parameter. Here is the method: We would like to know whether the model can reproduce the targeted pattern for different evaporation rates. We divide the parameter interval into `nX` intervals of the same size, and we apply a genetic algorithm to search values for other parameters (the ants model only takes 2 parameters, so that the dispersal parameter is the only one to be varied), which, as previously for the calibration, minimize the distance between the measurements produced by the model and the ones observed experimentally. In the calibration case, we kept the best individuals of the population whatever their parameter values. This time, we still keep the best individuals but we now guarantee to keep at least one for each interval division of the profiled parameter, that is the evaporation parameter. Then, we do the same operation with the dispersal parameter.

To set a profile for a given parameter in OpenMOLE, the GenomeProfile evolutionary method is used:


    val evolution =
       GenomeProfile (
         x = 0,
         nX = 20,
         inputs =
            Seq(
              diffusion -> (0.0, 99.0),
              evaporation -> (0.0, 99.0)),
         termination = 100 hours,
         objective = aggregatedFitness,
         reevaluate = 0.01
       )

The arguments `inputs`, `termination`, `objective` and `reevaluate` have the same role as in calibration. The argument `objective` is this time not a sequence but a single objective to minimize. The argument `x` specifies the index of the parameter to be profiled, i.e. its position within the `inputs` sequence , indexing starting at 0. `nX` is as explained before the size of the discretization of its range.

As for any evolutionary method, we need for each profile to create the OpenMOLE puzzle to execute it. We define a function returning the puzzle associated to a given parameter `parameter` and use it to assemble all pieces into a common puzzle, as follows :


    def profile(parameter: Int) = {
        val evolution =
           GenomeProfile (
             x = parameter,
             nX = 20,
             inputs =
                Seq(
                  diffusion -> (0.0, 99.0),
                  evaporation -> (0.0, 99.0)),
             termination = 100 hours,
             objective = aggregatedFitness,
             reevaluate = 0.01
           )

        val (puzzle, ga) = SteadyGA(evolution)(replicateModel, 40)
        val savePopulationHook = SavePopulationHook(ga, workDirectory / ("results/" + parameter.toString))
        val display = DisplayHook("Generation ${" + ga.generation.name + "}")
        (puzzle hook savePopulationHook hook display)
    }

    //assemblage
    val firstCapsule = Capsule(EmptyTask())
    val profiles = (0 until 2).map(profile)
    profiles.map(firstCapsule -- _).reduce(_ + _)

We obtain the following profiles :

![](ants_profiles/profile_diffusion.png) 

![](ants_profiles/profile_evaporation.png) 

Except for values below 10, the model is able to reproduce rather accurately experimental measures for any value of diffusion rate. A refined profile within the interval \[0;20\] may be useful to have a more precise idea. Concerning the evaporation parameter, model performance is on the contrary strongly sensitive, as values over 10 lead to a strong increase in minimal fit. When running the model with a diffusion rate of 21 and evaporation rate of 15, we observe that ants are not able to build a pheromone path enough stable between the nest and furthest food pile, what increases the time needed to exploit it in a considerable way.


![](img/ants_evaporation_15.png) 

Sensitivity analysis: Robustness of a calibration
-------------------------------------------------

The last method presented here aims to evaluate the robustness of a model calibration. We mean by a robust calibration that small variations of optimal parameters do not strongly change model behavior. In other words there should be no discontinuity in model indicators in a reasonable region around the optimal point. As a consequence, if parameter values are restricted to given regions of the parameter space, we expect the model to have roughly the same behavior within each region, especially within the region around the calibrated point.

Let's suppose for instance, that we can measure the parameter values directly in the data. Let's also admit that we can establish a confidence interval for each parameter. We want to be sure that, as long as the parameter values remain in their respective intervals, the model keep the same behavior. This step is important when we try to use the model as a predictable model. If the model produces behaviors very different in the considered intervals, the parameters responsible for this variation has to be found and has to be measured with more accuracy to reduce the confidence interval.

This issue can be tackled using the PSE algorithm again, by running the above example with the desired confidence intervals for each parameter. The algorithm will aim to diversity of outputs within these interval, and the unveiling of significantly different patterns will imply that the model is sensitive to some parameters within the considered region. One must then either narrow parameters bounds again, or stay cautious on conclusions obtained through the calibrated model.

Conclusion
----------

The methods developed here are insights into a pattern-oriented of complex systems modeling and simulation, in the sense of patterns produced as outputs of model simulations. When data on internal mechanisms of a system or causing an emerging phenomenon are missing, because they are not directly observable for example, they can be formulated into algorithmic interpretations (models of simulation), which are candidate as explanations of the phenomenon. The method we developed here are various ways to verify wether these propositions are able to reproduce (regarding given objectives) the phenomenon they aim to explain, to test their predictive and explicative capabilities and to analyze the role of each parameter in their dynamic.


[![http://creativecommons.org/licenses/by-sa/4.0/](license-by-sa.png)](http://creativecommons.org/licenses/by-sa/4.0/)

*This text by Guillaume Chérel is under a Creative Commons Attribution - Share alike 4.0 International license. To obtain a copy of this license, please visit http://creativecommons.org/licenses/by-sa/4.0/ or send an inquiry to Creative Commons, 444 Castro Street, Suite 900, Mountain View, California, 94041, USA.*
