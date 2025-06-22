/****************************************************************************************
Statistical Programming Exam - Stata Implementation
Author: Paolo Machaca
Fecha: 22-06-2025
Objetivo: Manipulación, limpieza, visualización y análisis de datos
Software: Stata 17
****************************************************************************************/

/***************************************
* SECTION 1: HANDLING DATA
****************************************/

* 1.1 Cargar y combinar las bases de datos
* Se cargan los datos de línea base (maindata) y seguimiento largo plazo (longterm),
* añadiendo una variable "source" que indica el origen de cada observación.

import excel "C:\Users\Paolo\Desktop\maindata.xls", sheet("Sheet1") firstrow clear
    gen source = "main"
    save "C:\Users\Paolo\Desktop\maindata_temp.dta", replace

import excel "C:\Users\Paolo\Desktop\longterm.xls", sheet("Sheet1") firstrow clear
    gen source = "longterm"
    append using "C:\Users\Paolo\Desktop\maindata_temp.dta"

* 1.2 Preservar variable sensible
* La variable "social_security" contiene información identificable, por lo que se guarda
* en un archivo separado para proteger la confidencialidad.

preserve
    keep folio social_security
    save "C:\Users\Paolo\Desktop\social_security.dta", replace
restore

* 1.3 Eliminar variable sensible de la base principal
drop social_security
save "C:\Users\Paolo\Desktop\combined_nossn.dta", replace


/***************************************
* SECTION 2: CLEANING DATA
****************************************/

* 2.1 Etiquetado de la variable sector (sec)
* Se asignan etiquetas a los valores de la variable sec según el codebook.

label define sec 1 "industry" 2 "commerce" 3 "service" -997 "don't know"
label values sec sec

* 2.2 Crear cuartiles de costos 2008 válidos
* Se filtran valores válidos (>= 0) de total_costs_2008 y se generan cuartiles.

gen total_costs_2008_valid = .
replace total_costs_2008_valid = total_costs_2008 if inrange(total_costs_2008, 0, .)
xtile costs_quartile_2008 = total_costs_2008_valid, nq(4)

* 2.3 Revisión y limpieza de valores faltantes en total_profits_2008
* a) Identificación de valores negativos como codificación de missing

tab total_profits_2008 if total_profits_2008 < 0

* b) Comentario: si los valores -997, -998, -999 no se tratan como missing,
* distorsionan estadísticas como la media y regresiones, dando lugar a sesgo.

* c) Reemplazo de códigos de missing por el valor real de missing (.)

replace total_profits_2008 = . if inlist(total_profits_2008, -997, -998, -999)

* 2.4 Guardar base lista para análisis
save "C:\Users\Paolo\Desktop\finaldata.dta", replace


/***************************************
* SECTION 3: VISUALIZING DATA
****************************************/

* 3.1 Establecer rutas portables
* Se define una macro local y global para mejorar la portabilidad del código.

local path "C:\Users\Paolo\Desktop"
global path "C:/Users/Paolo/Desktop"
use "`path'\finaldata.dta", clear

* 3.2 Test de balance en línea base
* Se estima la diferencia promedio en variables de empleo según treatment en followup == 0

foreach i in full_time_employees part_time_employees seasonal_employees {
    reg `i' treatment if followup == 0
    eststo `i'
}

* Exportar resultados de balance a archivo .tex para inclusión en informe LaTeX

esttab using "${path}/BalanceTestLatex1.tex", se nodepvar nonumber ///
    star(* 0.10 ** 0.05 *** 0.01) cells(b(star fmt(%9.3f)) se(par)) ///
    stats(N, fmt(%9.0f)) legend collabels(none) varlabels(_cons Constant)

* 3.3 Estandarización de número de solicitudes de préstamo
* Se crea una variable estándar y se grafica su distribución

sum loan_bank_number if loan_bank_number >= 0, detail
gen std_numb_bankloan2 = (loan_bank_number - r(mean)) / r(sd)

histogram std_numb_bankloan2, normal width(0.5) frequency ///
    title("Distribución estandarizada de solicitudes de préstamo")


/***************************************
* SECTION 4: ANALYZING DATA
****************************************/

* 4.1 Inclusión de todas las categorías de año
* Para evitar omisión de dummies por colinealidad, se crean variables con "tab, gen"

tab followup, gen(fup)
reg profits treatment fup1 fup2  // fup0 como base

* Exportar esta regresión a LaTeX
eststo profits_model
esttab profits_model using "${path}/profits_model1.tex", se nodepvar nonumber ///
    star(* 0.10 ** 0.05 *** 0.01) cells(b(star fmt(%9.3f)) se(par)) stats(N r2, fmt(%9.0f %6.3f)) label

* 4.2 LATE: modelos IV con y sin controles
* Se estima el efecto causal del programa usando treatment como instrumento

global controls "trmrk sec total_employees"

ivregress 2sls sales (in_program = treatment), robust
eststo model_iv_nocontrols

ivregress 2sls sales $controls (in_program = treatment), robust
eststo model_iv_controls

* Exportar modelos IV
esttab model_iv_nocontrols model_iv_controls using "${path}/ivreg_late_models1.tex", ///
    se nodepvar nonumber star(* 0.10 ** 0.05 *** 0.01) ///
    cells(b(star fmt(%9.3f)) se(par)) stats(N r2, fmt(%9.0f %6.3f)) label


/***************************************
* SECTION 5: INTERPRETING RESULTS
****************************************/

* 5.1 Justificación del instrumento
* El tratamiento fue asignado aleatoriamente, mientras que in_program puede estar
* correlacionado con características no observadas como motivación o experiencia.

* 5.2 Evaluar significancia del tratamiento en distintos outcomes
global outcome_list sales profits trmrk
foreach var of global outcome_list {
    reg `var' treatment
    eststo `var'_reg
    display "Efecto del tratamiento en `var':"
    test treatment
    display "p-valor para `var': " %6.4f r(p)
}

* Exportar tabla de efectos del tratamiento
esttab sales_reg profits_reg trmrk_reg using "${path}/treatment_effects_outcomes1.tex", ///
    se nodepvar nonumber star(* 0.10 ** 0.05 *** 0.01) ///
    cells(b(star fmt(%9.3f)) se(par)) stats(N r2, fmt(%9.0f %6.3f)) label

* 5.3 Interacción género x etnicidad sin constante
* Cada coeficiente representa el promedio de ventas para un grupo definido por género y etnicidad.

regress sales i.male##i.indigenous, noconstant

* Interpretación:
* - 1.male#1.indigenous: hombres indígenas
* - 1.male#0.indigenous: hombres no indígenas
* - 0.male#1.indigenous: mujeres indígenas
* - 0.male#0.indigenous: mujeres no indígenas (referencia implícita si hubiera constante)
