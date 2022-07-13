# WSS_UQ
Uncertainty Quantification on gjackland's Weight-Scale-Shift (WSS) model for Covid epidemic modelling

## Background on model
The WSS model takes current real-world case data and (among other things) predicts current and future R numbers (often days/weeks before other R_no predictions which are based on hosp./deaths) and from there predicts hospitalisations and deaths for a number of days into the future (we've imposed a cut off at 54 days after the last day of case data) across 10 regions in the UK. In this framework we are only considering the number of deaths in one region.

## Uncertainty Quantification
The WSS model is expensive to run (around 5 minutes per model run) so to get a representation across the entire input space would take an unfeasible amount of time. Therefore an alternative is to create a statistical approximation to the model (in this case a Gaussian process emulator) which is trained by a small number of model runs (usually 10p where p is no. of input dims.) and can quantify the uncertainty in the areas of input space which haven't been trained.

## Framework for UQ with WSS model
### Inputs
This model can be split into two parts: the first being using the case data to predict the R number and the second part being using those R numbers to predict future case, hospitalisations and deaths. As the first part can be represented by an R number then by treating this as a variable we can span all possible case data.

Knowing this we can now bring together all the inputs:
- this model is compartmental with those being: 1) Mild; 2) ILI (influenza-like illness); 3) SARI (severe acute respiratory illness); 4) Critial; 5) Death; 6) Recovery and 7) Critical Recovery. There are 8 model parameters which is the logmean number of days it takes someone to go between the following compartments: 1->6, 2->6, 2->3, 3->6, 3->5, 3->4, 4->7, 4->5 and 7->6 with 4->7 and 4->5 having the same logmean. 
- 5 coefficients that dictate the severity and transmissability of the alpha, delta and omicron variants (except tranmissability of omicron)
- vaccine efficacy
- rate at which R number decays back to 1
- 2 beta parameters that define case-fatality-ratio (CFR) for each age-group.
- R_no from first part

### Outputs
We have imposed a cut off of 54 days after the R number has been set. More specifically we are looking at 01/06/22 - 25/07/22 (note that these dates can be changed by modifying the 'startdate' and 'enddate' parameters in 'getParams_ensemble.R', as it stands the enddate is 01/06/22 as its the last day of data, just set startdate to be an arbitrarily long time before that). So to be clear: one model run will take 18 inputs and its output will be a timeseries of 55 days (days 0-54).

### Principal Component Analysis
Gaussian process emulation is designed to only emulate one output therefore for p outputs, p emulators would be built. For a timeseries of 55 days this would mean building 55 emulators which is inefficient for two reasons: 1) the compuational power it would take to build these emulators would involve inverting 55 10px10p matrices could be better spent elsewhere and 2) the outputs are not independent from each other so one would need to propose how the outputs interact with each other which introduce further uncertainty. An alternative is to use Principal Component Analysis (PCA).

We are able to represent the output timeseries using a number of orthogonal basis functions and weights. To determine these, a large amount of timeseries outputs (from the training data) are collated and from there a function is constructed which explains the most amount of variance in the output; this is the first principal component. This is repeated until the desired amount of variance has been explained. This drastically reduces the number of outputs to deal with as the number of weights << original length of output using this process. In summary: after this process we have m basis functions (where the basis functions are the same length as the timeseries) with m weights representing each timeseries output. Note that when representing another timeseries output; the weights change but the basis functions stay the same.

### Emulation and Prediction
After conducting the PCA we now have a far smaller number of outputs with which we can build emulators for - these outputs now being the weights for the basis functions. We fit an emulator to each of these weights (which are now independent from each other as their respective basis functions are orthogonal) which enables us to construct a distribution of timeseries' with a mean and confidence intervals.

## Implementing this framework
1. Download the model from https://github.com/gjackland/WSS
2. In the same folder as the 'covid_trimmed.R', 'Regional.R' etc files, add in the following files: 'getParams_ensemble.R', 'covid_trimmed_oneregion.R', 'Regional_oneregion.R', 'CompartmentFunction_ensemble.R', 'Workflow_ensemble.R' and the folder 'csv_files'. These adapt the code slightly to allow for running ensembles.
3. Run the ensemble using 'model_runs.R'. This does model runs for design and validation purposes. Parameter ranges were agreed with the model developer.
4. Conduct PCA by running 'PCA.R'. This requires some interaction (lines 157, 161, 167 and 171) to tailor PCA by removing some outlying design points.
5. We can emulate the weights of the PCA and produce probability distribution of timeseries with mean, +-2SD and individual draws using the file 'emulation_pcs.R'. One can compare the approximation with a validation timeseries using line 130.
