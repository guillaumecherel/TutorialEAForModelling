# Calibration, validation et sensitivity analysis of complex systems models with OpenMOLE

Guillaume Chérel, 2015-10-23

Translated from the french by Guillaume Chérel, Mathieu Leclaire, Juste Raimbault, Julien Perret.

[![http://creativecommons.org/licenses/by-sa/4.0/](license-by-sa.png)](http://creativecommons.org/licenses/by-sa/4.0/) Guillaume Chérel, 2015

This text is licensed under the Creative Commons Attribution-ShareAlike 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/4.0/.

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

When this number stabilizes, PSE does not make new discories anymore. One has to be careful when interpreting this. Indeed, the absence of new discoveries can mean that all the patterns that the model can produce have been discovered, but it is also possible that other patterns exist but that PSE could not reach them.

The following figure shows the patterns discovered by PSE when we interrupted the exploration.

![](ants_pse/patterns.png) 

<!---

La première observation que l'on peut faire est que dans tous les motifs découverts, la source de nourriture la plus proche a été épuisée avant la source de nourriture la plus éloignée. D'autre part, il semble y avoir une borne minimum et maximum pour le temps au cours duquel la première source de nourriture est consommée.

Ces trois observations nous donne autant de point de départ de nouvelles réflexions sur le comportement collectif des fourmis. Par exemple, l'exploitation en priorité des sources de nourritures plus proches est-elle systématique? Pourrait-il exister des espèces de fourmis qui exploiterait d'abord des sources de nourriture plus éloignées que d'autres? Si l'on trouvait une telle espèce, il faudrait se demander par quel mécanisme cela est possible, et revoir le modèle pour qu'il puisse en rendre compte. Ceci illustre comment la découverte des différents comportements que peut produire le modèle peut nous amener à formuler de nouvelles hypothèses sur le système étudié, à les tester, et à réviser le modèle, en faisant avancer notre compréhension du phénomène.

Pourquoi ne pas simplement échantillonner l'espace des paramètres pour connaître les différents comportements possibles du modèle, avec des méthodes d'échantillonnage bien connues comme le LHS? Dans une expérience avec un modèle de déplacement collectif à 5 paramètres, nous avons comparé les performances de PSE et de 3 échantillonnages dans l'espace de paramètres: LHS, Sobol et une grille régulière. Les résultats représentés dans les deux figures suivantes ont montré que l'échantillonnage de l'espace de paramètres, même lorsqu'il a de bonne propriétés de couverture de l'espace comme le LHS et Sobol, peut passer à côté de nombreux motifs, et qu'il faut préférer une méthode adaptative comme PSE qui oriente la recherche en fonction des découvertes faites au cours de celle-ci. La figure suivante montre les comportements découverts par la méthode proposée (PSE pour Pattern Space Exploration), par un échantillonnage LHS et par un échantillonnage en grille régulière.

<img src="img/flockingpatternsallmethods.png" width="600" />

Chaque point représente un comportement du modèle découvert. Les comportements sont décrits en 2 dimensions: la vélocité moyenne des particules qui se déplacent et leur diffusion relative (vers 1, elles se déplacent les unes par rapport aux autres, à 0 elles restent fixes les unes par rapport aux autres et vers -1 elles se rapprochent les unes des autres).
-->

The following figure allows to compare PSE to other sampling methods in terms of efficiency.

<img src="img/flockingmethodcomparisonsimple.png" width="350" />

Sensitivity analysis: Profiles
--------------------------------

*Linked OpenMOLE script: [ants\_profiles/ants\_profiles.oms](ants_profiles/ants_profiles.oms)*

*Article: Reuillon R., Schmitt C., De Aldama R., Mouret J.-B., 2015, "A New Method to Evaluate Simulation Models: The Calibration Profile (CP) Algorithm", JASSS : Journal of Artificial Societies and Social Simulation, Vol. 18, Issue 1, <http://jasss.soc.surrey.ac.uk/18/1/12.html>*

The method we now present aims at understanding better how the model works in focusing on the impact of the different parameters of the model. In our Anthills example, we previously calibrated the model to enforce it to reproduce fake experimental measurements. We would like to knowe whether the model can reproduce this pattern for other parameter values. It is possible for instance, that a parameter is crucial and yet the model cannot reproduce the experimental measurements for a different value other than the one found with the calibration. It is also possible, on the contrary, that another parameter is not essential at all, that is, the model can reproduce the experimental measurements whatever its value. To establish the relevancy of our model parameter, we will set the parameters profiles for the model and for the targeted pattern, as follows:

Commençons par établir le profil du paramètre d'évaporation. La méthode est la suivante. On voudrait savoir si le modèle peut reproduire le motif visé pour différents taux d'évaporation. On divise l'intervalle du paramètre en `nX` intervalles de même taille, et on utilise un algorithme génétique pour rechercher des valeurs des autres paramètres (dans le modèle de fourmi il n'y en a que deux, c'est donc seulement le paramètre de dispersion qui va varier) qui, comme précédemment pour la calibration, minimisent la distance entre les mesures produites par le modèle en simulation et celles observées expérimentalement. Dans le cas de la calibration, on gardait les meilleurs individus de la population quelles que soient leurs valeurs de paramètre. Cette fois, on garde aussi les meilleurs individus, mais en s'assurant d'en garder au moins un pour chaque division de l'intervalle du paramètre dont on établit le profile, c'est-à-dire le taux d'évaporation. Une fois fait, on recommence pour l'autre paramètre, celui de dispersion.

Pour établir un profil dans OpenMOLE pour un paramètre donné, on utilise la méthode d'évolution GenomeProfile:

    val evolution =
       GenomeProfile (
         x = 0,
         nX = 10,
         inputs = 
            Seq(
              diffusion -> (0.0, 99.0), 
              evaporation -> (0.0, 99.0)),
         termination = 100 hours,
         objective = aggregatedFitness,
         reevaluate = 0.01
       )

Les arguments `inputs`, `termination`, `objective` et `reevaluate` ont la même signification que pour la calibration. L'argument `objective`, cette fois, ne prend pas une séquence mais un seul objectif, une valeur à minimiser. L'argument `x` spécifie le numéro du paramètre dont on veut établir le profile. Ce numéro commence à 0, et correspond à la position du paramètre dans la séquence donnée à `inputs`. Enfin, l'argument `nX` contrôle en combien d'intervalle on veut diviser l'intervalle du paramètre profilé.

Comme pour chaque méthode d'évolution vue précédemment, il faut construire pour chaque profile le morceau de puzzle OpenMOLE qui va permettre de l'exécuter. On définit une fonction qui construit le puzzle associé au profil du paramètre `i`, puis on assemble tous les puzzles en un puzzle commun, comme ci-dessous:

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

Voici les profiles obtenus pour chaque paramètre:

![](ants_profiles/profile_diffusion.png) 

![](ants_profiles/profile_evaporation.png) 

Le modèle semble pouvoir reproduire assez fidèlement les mesures expérimentales indépendamment du taux de diffusion, sauf peut-être lorsqu'il descend en dessous de 10. On pourrait relancer un profile sur l'intervalle \[0;20\] pour en avoir une idée plus précise. En revanche, la reproduction des mesures expérimentales par le modèle est très sensible au paramètre d'évaporation. Le comportement du modèle s'en éloigne brusquement dès que le taux d'évaporation dépasse 10. Si on fait tourner le modèle avec un taux de diffusion de 21 et un taux d'évaporation de 15, on se rend compte que les fourmis ne peuvent plus construire un chemin de phéromone suffisamment stable entre le nid et la source de nourriture la plus éloignée, ce qui augmente considérablement le temps qu'il leur faut pour l'exploiter.

![](img/ants_evaporation_15.png) 

Analyse de sensibilité: Robustesse d'un calibrage
-------------------------------------------------

La dernière méthode vise à évaluer la robustesse du calibrage d'un modèle. Un calibrage robuste signifie que de petites variations des valeurs de paramètres ne produisent pas de changement important du comportement du modèle en simulation. En conséquence, on peut prédire que tant que l'on restreint les valeurs de paramètres à des intervalles donnés, le modèle donnera toujours globalement le même comportement.

Supposons par exemple que l'on puisse mesurer les valeurs de paramètres directement dans les données. Admettons que l'on puisse établir un intervalle de confiance pour chaque paramètre. On veut s'assurer que, tant que les valeurs de paramètres restent dans leur intervalle respectif, le modèle conserve toujours globalement le même comportement. Cette étape est importante lorsque l'on cherche à tirer des prédictions d'un modèle. Si le modèle produits des comportements très variés dans les intervalles considérés, alors il faut trouver quels paramètres sont responsables de cette variation et tenter de les mesurer avec plus de précision pour réduire l'intervalle de confiance.

On peut à nouveau utiliser PSE pour répondre à cette question. Il suffit de reprendre l'exemple ci-dessus en changeant les intervalle relatif à chaque paramètre pour les remplacer par les intervalles de confiance désirés. L'algorithme va alors chercher les différents motifs que peut générer le modèle dans cet intervalle. S'il découvre des motifs différents, c'est que le modèle est sensible à certains paramètres dans les intervalles considérés, et qu'il faut être prudent quant aux conclusions que l'on tire à partir du modèle calibré.

Conclusion
----------

Les méthodes présentées ici dessinent une approche de la modélisation informatique de systèmes complexes centrée sur les motifs que produisent les modèles en simulation. Lorsque l'on manque de données sur les mécanismes qui produisent un phénomène, par exemple parce qu'ils sont difficiles à observer directement, on peut en fournir une interprétation sous forme algorithmique. Cette interprétation est une proposition d'explication du phénomène. Les méthodes proposées ci-dessus sont autant de manières de vérifier que ces propositions sont capables de reproduire le phénomène qu'elles visent à expliquer, de mettre à l'épreuve leurs capacités prédictives et explicatives, et d'analyser le rôle des différents paramètres dans leur dynamique.

[![http://creativecommons.org/licenses/by-sa/4.0/](license-by-sa.png)](http://creativecommons.org/licenses/by-sa/4.0/)

*Ce texte par Guillaume Chérel est sous licence Creative Commons Attribution - Partage dans les Mêmes Conditions 4.0 International. Pour accéder à une copie de cette licence, merci de vous rendre à l'adresse suivante http://creativecommons.org/licenses/by-sa/4.0/ ou envoyez un courrier à Creative Commons, 444 Castro Street, Suite 900, Mountain View, California, 94041, USA.*
