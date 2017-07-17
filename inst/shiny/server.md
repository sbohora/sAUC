Below is the `server.R` code

```r
library(shiny)
library(dplyr)
library(magrittr)
library(tidyr)
library(ggplot2)
library(DT)

shinyServer(function(input, output){
  output$menu <- renderMenu({
    sidebarMenu(
      menuItem("Menu Item", icon =icon("calendar"))
    )
  })

  # Create reactive to read data
  data <- reactive({
    input_file <- input$file
    if(is.null(input_file)){return()}
    read.table(
      file = input_file$datapath,
      sep = input$sep,
      header = input$header,
      stringsAsFactors = input$string_factors
    )
  })

  #The following set of functions populate the column selectors
  output$choose_response <- renderUI({
    df <- data()
    if (is.null(df)) return(NULL)

    items=names(df)
    names(items)=items
    selectInput(
      inputId = "response",
      label = "Choose response:",
      choices = items)
  })

  output$choose_group <- renderUI({
    df <- data()
    if (is.null(df)) return(NULL)

    items=names(df)
    names(items)=items
    selectInput(
      inputId = "group_var",
      label = "Choose group:",
      choices  = names(data())[!names(data()) %in% input$response],
      selected = names(data())[!names(data()) %in% input$response][1])
  })

  output$independent <- renderUI({
  checkboxGroupInput(inputId = "independent",
                     label =  "Independent Variables:",
                     choices = names(data())[!names(data()) %in% input$response],
                     selected = names(data())[!names(data()) %in% input$response][2])
  })

  output$model_result <- renderDataTable({
    if (is.null(data())) return(NULL)
    ds <- data()
    cov_variables <- c(input$independent,input$group_var)
    ds[, cov_variables] <- lapply(ds[, cov_variables], function(x) factor(x))
    res <- sAUC::sAUC(x = as.formula(paste(input$response," ~ ",paste(input$independent,collapse="+"))),
               treatment_group = input$group_var, data = ds)

    DT::datatable(as.data.frame(res$"Model summary"),
                caption = htmltools::tags$caption(
                  style = "font-size:105%",
                  strong(paste('Model results'))))
  })

  # Display orginal data
  output$show_input_file <- renderTable({
    if(is.null(data())){return()}
    input$file
  })

    # Display orginal data
  output$show_data <- renderDataTable({
    if(is.null(data())){return()}
    data()
  })

  # Display summary of the original data
  output$summaryy <- renderDataTable({
    ds <- data()
    # numeric_columns <- names(ds)[sapply(ds, function(x) is.numeric(x))]
    if(is.null(ds)){return()}
    summary_table <- as.data.frame(round(psych::describe(ds)[-1]))
    names(summary_table) <- Hmisc::capitalize(names(summary_table))
    datatable(summary_table,
              caption = htmltools::tags$caption(
                style = "font-size:200%",
                htmltools::strong(paste("Table 1: Descriptive summary"))),
              rownames = TRUE)
  })

  output$plot_data <- renderPlot({
    psych::pairs.panels(data())
  })

  output$describe_file <- renderUI({
    if (is.null(data())){
      h3("Data are not read yet. Please do so now if you'd like to run Semiparametric AUC Regression model.", style = "color:red")
    } else {
      tabsetPanel(
        tabPanel(
          title = "About file",
          tableOutput("show_input_file")),
        tabPanel(
          title = "Data",
          dataTableOutput("show_data")),
        tabPanel(
          title = "Summary",
          dataTableOutput("summaryy")),
        tabPanel(
          title = "Plots",
          plotOutput("plot_data"))
      )
    }
  })

  result_of_simulate <- reactive({
    iter <- input$realization
    m <- input$number_treatment
    p <- input$number_control
    b0 <- input$b0
    b1 <- input$b1
    b2 <- input$b2
    sAUC::simulate_one_predictor(iter = iter, m = m, p = p, b0 = b0, b1 = b1, b2 = b2)
  })

  output$result1 <- DT::renderDataTable({
    result_simulate <- result_of_simulate()
    df <- (as.data.frame(cbind(result_simulate$meanbeta, result_simulate$meanvar, result_simulate$meansd, result_simulate$ci_betass, result_simulate$all_coverage, result_simulate$iter)))
    names(df) <- c("Beta Estimates", "Variance of Beta", "S.E. of Beta","Confidence Interval on Beta", "Coverage Probability", "Iterations")
    dt <- DT::datatable(
      df,
      caption = htmltools::tags$caption(
        style = "font-size:150%",
        'Table 1. Results of the Simulation on sAUC with one discrete covariate'),
      rownames = c("B0", "B1", "B2"))
  })

  output$result_plot_beta <- renderPlot({
    simulated_betas <- result_of_simulate()
    dddd <- as.data.frame(simulated_betas$m_betas)
    data_long <- gather(dddd, Parameter, values, factor_key=TRUE)
    data_long$Parameter <- with(data_long, ifelse(Parameter == "V1","0",
                                                  ifelse(Parameter =="V2","1", "2")))
    mu <- data_long %>%
      dplyr::group_by(Parameter) %>%
      dplyr::summarize(mean_beta = mean(values)) %>% as.data.frame()

    # Create normal curve to overlay to plot
    # calculate mean and sd by group
    stats <- aggregate(values~Parameter, data_long, function(x) c(mean=mean(x), sd=sd(x)))
    stats <- data.frame(Parameter=stats[,1],stats[,2])
    x <- with(data_long, seq(min(values), max(values), len=100))
    dfn <- do.call(rbind,lapply(1:nrow(stats),
                                function(i) with(stats[i,],data.frame(Parameter, x, y=dnorm(x,mean=mean,sd=sd)))))

    # Change colors by groups
    ggplot(data_long, aes(x=values, color=Parameter, fill=Parameter)) +
      geom_histogram(aes(y=..density..), position="identity", alpha=0.7, bins = 50) +
      geom_density(alpha=0.6, size = 0.9, adjust = 0.6) +
      facet_grid(.~Parameter, labeller = label_bquote(cols = beta[.(Parameter)])) +
      geom_vline(data=mu, aes(xintercept=mean_beta, color=Parameter),linetype="dashed") +
      scale_color_manual(values=c("blue", "red", "maroon")) +
      # scale_fill_manual(values=c("#999999", "#E69F00", "#56B4E9"))+
      labs(x="Estimates", y = "Density") +
      theme_classic() +
      theme(text = element_text(size=20)) +
      theme(legend.position="none") +
      geom_line(data=dfn, aes(x, y), alpha = 0.3, size= 1.2, colour = "black")
  })

  output$hist_gram <- renderPlot({
    hist(rnorm(10))
  })
})
```