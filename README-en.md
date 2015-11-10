# Calibration, validation et sensitivity analysis of complex systems models with OpenMOLE

Guillaume Chérel, 2015-10-23

Translated from the french by Guillaume Chérel, Mathieu Leclaire, Juste Raimbault, Julien Perret.

[![http://creativecommons.org/licenses/by-sa/4.0/](license-by-sa.png)](http://creativecommons.org/licenses/by-sa/4.0/) Guillaume Chérel, 2015

This text is licensed under the Creative Commons Attribution-ShareAlike 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/4.0/.


Sensitivity analysis: Profiles
--------------------------------

*Linked OpenMOLE script: [ants\_profiles/ants\_profiles.oms](ants_profiles/ants_profiles.oms)*

*Article: Reuillon R., Schmitt C., De Aldama R., Mouret J.-B., 2015, "A New Method to Evaluate Simulation Models: The Calibration Profile (CP) Algorithm", JASSS : Journal of Artificial Societies and Social Simulation, Vol. 18, Issue 1, <http://jasss.soc.surrey.ac.uk/18/1/12.html>*

The method we now present aims at understanding better how the model works in focusing on the impact of the different parameters of the model. In our Anthills example, we previously calibrated the model to enforce it to reproduce fake experimental measurements. We would like to know whether the model can reproduce this pattern for other parameter values. It is possible for instance, that a parameter is crucial and yet the model cannot reproduce the experimental measurements for a different value other than the one found with the calibration. It is also possible, on the contrary, that another parameter is not essential at all, that is, the model can reproduce the experimental measurements whatever its value. To establish the relevancy of our model parameter, we will set the parameters profiles for the model and for the targeted pattern, as follows:

We first establish the profil of the evaporation parameter. Here is the method: We would like to know whether the model can reproduce the targeted pattern for different evaporation rates. We divide the parameter interval into `nX` intervals of the same size, and we apply a genetic algorithm to search values for other parameters (the ants model only takes 2 parameters, so that the dispersal parameter is the only one to be varied), which, as previously for the calibration, minimize the distance between the measurements produced by the model and the ones observed experimentally. In the calibration case, we kept the best individals of the population whatever their parameter values. This time, we still keep the best individuals but we now guarantee to keep at least one for each interval division of the profiled parameter, that is the evaporation parameter. Then, we do the same operation with the dispersal parameter.

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

The arguments `inputs`, `termination`, `objective` and `reevaluate` have the same meaning as in the calibration. The argument `objective` is no more a sequence of objectives, it now a single objective value to minimize. The argument `x` specifies the number of the profiled parameter. This number starts at 0. It is the position of the parameter in the `inputs` sequence. Finaly, the argument `nX` is the number of required divisions for the profiled parameter.

As it is the case for each evolutianary method seen before, we have to build for each profile a piece of OpenMOLE puzzle (in order to execute it). We define a function, which builds the puzzle associated to the profile of the parameter `i`, then we merge all the puzzles into a common puzzle as below:    

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

Here are the profiles obtained for each parameter:

![](ants_profiles/profile_diffusion.png) 

![](ants_profiles/profile_evaporation.png) 

The model seems to be abble to reproduce fairly accurately the experimental measurements whatever the diffusion rate, except peharps when it is lower than 10. We could start again the process on the interval \[0;20\] to be have more accurate idea in this region. However, the reproduction of the experimental measurements by the model is very sensitive to the evaporation parameter. The behaviour of the model is very far from the experiments if the evaporation rate exceeds 10. If the model is run with a diffusion rate of 21 and an evaporation rate of 15, we realize the ants cannot build any more a pheromone path stable enought between the nest and the farthest food spot, which increases dramatically the required time to operate it.

![](img/ants_evaporation_15.png) 

Sensitivity analysis: Robustness of a calibration
-------------------------------------------------

The last method aims at evaluating the robustness of a model calibration. A robust calibration means that small small variations of the parameter values do not produce major variations on the model outputs. Consequently, we can predict that if we restrict the values of the parameters to given intervals, the model will keep globaly the same behaviour.

Let's suppose for instance, that we can measure the parameter values directly in the data. Let's also admit that we can establish a confidence interval for each parameter. We want to be sure that, as long as the parameter values remain in their respective intervals, the model keep the same behaviour. This step is important when we try to use the model as a predictable model. If the model produces behaviours very different in the considered intervals, the parameters responsible for this variation has to be found and has to be measured with more accuracy to reduce the confidence interval.

We can use PSE again to answer this question. We only need to use the previous example and to change the intervals relative to each parameter in order to replace them by the required confidence intervals. The algorithm will then search the different patterns that the model can produce in this interval. If different patterns are discovered, it means the model is sensitive to some parameters in the considered intervals and we must be careful to the conclusions we will draw from the calibrated model.  

Conclusion
----------
The methods presented here are a tentative to approach the numerical modeling in complex systems. They focus on the patterns that the model can produce. If data are missing on the mechanisms which produce a phenomena, for instance because they are difficult to observe directly, algorithmic interprétation can be provided. This interpretation is an explanation proposal of the phenomena. The methods proposed above are various ways to check if the model is able to reproduce the phenomena it wants to explain. They test the predictive and explanatory capacities of the model and analyse the role of its different parameters to its dynamics. 

[![http://creativecommons.org/licenses/by-sa/4.0/](license-by-sa.png)](http://creativecommons.org/licenses/by-sa/4.0/)

*Ce texte par Guillaume Chérel est sous licence Creative Commons Attribution - Partage dans les Mêmes Conditions 4.0 International. Pour accéder à une copie de cette licence, merci de vous rendre à l'adresse suivante http://creativecommons.org/licenses/by-sa/4.0/ ou envoyez un courrier à Creative Commons, 444 Castro Street, Suite 900, Mountain View, California, 94041, USA.*
