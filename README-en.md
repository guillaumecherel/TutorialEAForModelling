# Calibration, validation et sensitivity analysis of complex systems models with OpenMOLE

Guillaume Chérel, 2015-10-23

Translated from the french by Guillaume Chérel, Mathieu Leclaire, Juste Raimbault, Julien Perret.

[![http://creativecommons.org/licenses/by-sa/4.0/](license-by-sa.png)](http://creativecommons.org/licenses/by-sa/4.0/) Guillaume Chérel, 2015

This text is licensed under the Creative Commons Attribution-ShareAlike 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/4.0/.


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
