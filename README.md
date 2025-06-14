# Dynamic-pricing
system("apt-get update", intern = TRUE)
system("apt-get install -y libcurl4-openssl-dev libssl-dev libxml2-dev", intern = TRUE)
install.packages("tidyverse")
install.packages("ggplot2")
install.packages("dplyr")
install.packages("readr")
install.packages("lubridate")
install.packages("corrplot")
library(tidyverse)
library(ggplot2)
library(dplyr)
library(readr)
library(lubridate)
library(corrplot)
# Loading dataset
df <- read_csv("dynamic_pricing.csv")
glimpse(df)
colSums(is.na(df))  # This will give you the count of NAs per column
# Check for duplicate rows in the dataframe
duplicates <- df[duplicated(df), ]

# Check if there are any missing values in the duplicated rows
any(is.na(duplicates))
# Ordinal Encoding
# Label Encoding for Ordinal Variables
df <- df %>%
  mutate(Customer_Loyalty_Status = as.integer(factor(Customer_Loyalty_Status,
                 levels = c("Silver", "Regular", "Gold"), ordered = TRUE)),
         Vehicle_Type = as.integer(factor(Vehicle_Type,
                 levels = c("Economy", "Premium", "Luxury"), ordered = TRUE)))

# One-Hot Encoding for Nominal Variables
# Convert categorical columns to factors
df <- df %>%
  mutate(Location_Category = factor(Location_Category),
         Time_of_Booking = factor(Time_of_Booking))


df <- df %>%
  mutate(
    # Encoding for Time_of_Booking
    Time_of_Booking = factor(Time_of_Booking, levels = c("Morning", "Afternoon", "Evening", "Night"), ordered = TRUE) %>%
      as.integer(),

    # Encoding for Location_Category
    Location_Category = factor(Location_Category, levels = c("Rural", "Suburban", "Urban"), ordered = TRUE) %>%
      as.integer()
  )
# View result
head(df)
# Function to cap outliers using IQR
cap_outliers <- function(x) {
  Q1 <- quantile(x, 0.25, na.rm = TRUE)
  Q3 <- quantile(x, 0.75, na.rm = TRUE)
  IQR_val <- Q3 - Q1
  lower_bound <- Q1 - 1.5 * IQR_val
  upper_bound <- Q3 + 1.5 * IQR_val
  x[x < lower_bound] <- lower_bound
  x[x > upper_bound] <- upper_bound
  return(x)
}

# Apply outlier capping to all numeric columns
df[sapply(df, is.numeric)] <- lapply(df[sapply(df, is.numeric)], cap_outliers)
summary(df)
library(ggplot2)

# Get all numeric columns
numeric_columns <- names(df)[sapply(df, is.numeric)]

# Histograms
for (col in numeric_columns) {
  print(
    ggplot(df, aes_string(col)) +
      geom_histogram(fill = "skyblue", bins = 75, color = "black") +
      ggtitle(paste("Histogram of", col)) +
      theme_minimal()
  )
}

# Boxplots
for (col in numeric_columns) {
  print(
    ggplot(df, aes_string(x = "''", y = col)) +  # use empty string for x-axis label
      geom_boxplot(fill = "tomato") +
      ggtitle(paste("Boxplot of", col)) +
      xlab("") +
      theme_minimal()
  )
}
# Correlation matrix

library(corrplot)
options(repr.plot.width = 10, repr.plot.height = 10)  # Set width and height in inches
corrplot(
  cor(df, use = "complete.obs"),
  method = "color",
  type = "upper",
  tl.cex = 1.2,
  tl.srt = 60  # Rotate text labels to 45 degrees
)
write.csv(df, "cleaned_rideshare_data.csv", row.names = FALSE)
# Create new engineered features using existing columns
df$Ride_Density <- df$Number_of_Riders / df$Number_of_Drivers
df$Surge_Indicator <- ifelse(df$Number_of_Riders > 1.5 * df$Number_of_Drivers, 1, 0)

# Create Peak_Hours_Flag based on the ordered `Time_of_Booking` (1=Morning, 2=Afternoon, 3=Evening)
df$Peak_Hours_Flag <- ifelse(df$Time_of_Booking %in% c(1, 2, 3), 1, 0)  # Morning, Afternoon, Evening

# Now perform correlation analysis on all columns
correlation_matrix <- cor(df[, c(
  "Number_of_Riders", "Number_of_Drivers", "Location_Category",
  "Customer_Loyalty_Status", "Number_of_Past_Rides", "Average_Ratings",
  "Time_of_Booking", "Vehicle_Type", "Expected_Ride_Duration",
  "Historical_Cost_of_Ride", "Ride_Density", "Surge_Indicator",
  "Peak_Hours_Flag"
)], use = "complete.obs")

# Print the correlation matrix
print(correlation_matrix)

# Plot the correlation matrix using corrplot
library(corrplot)
corrplot(correlation_matrix, method = "color", type = "upper", tl.cex = 1.2)
install.packages("randomForest")
library(randomForest)
install.packages("xgboost")
library(xgboost)
# Splitting the data into training and test sets
set.seed(123)
train_index <- sample(1:nrow(df), 0.8 * nrow(df))
train_data <- df[train_index, ]
test_data <- df[-train_index, ]


# Random Forest Model
library(randomForest)
rf_model <- randomForest(Historical_Cost_of_Ride ~ Number_of_Riders + Number_of_Drivers + Location_Category + Customer_Loyalty_Status +
                           Number_of_Past_Rides + Average_Ratings + Time_of_Booking + Vehicle_Type + Expected_Ride_Duration +
                           Ride_Density + Surge_Indicator + Peak_Hours_Flag, data = train_data, ntree = 100)
rf_preds <- predict(rf_model, test_data)
rf_mae <- mean(abs(rf_preds - test_data$Historical_Cost_of_Ride))
rf_r2 <- cor(rf_preds, test_data$Historical_Cost_of_Ride)^2

# XGBoost Model
library(xgboost)
dtrain <- xgb.DMatrix(data = as.matrix(train_data[, c("Number_of_Riders", "Number_of_Drivers", "Location_Category",
                                                       "Customer_Loyalty_Status", "Number_of_Past_Rides", "Average_Ratings",
                                                       "Time_of_Booking", "Vehicle_Type", "Expected_Ride_Duration",
                                                       "Ride_Density", "Surge_Indicator", "Peak_Hours_Flag")]),
                      label = train_data$Historical_Cost_of_Ride)
dtest <- xgb.DMatrix(data = as.matrix(test_data[, c("Number_of_Riders", "Number_of_Drivers", "Location_Category",
                                                      "Customer_Loyalty_Status", "Number_of_Past_Rides", "Average_Ratings",
                                                      "Time_of_Booking", "Vehicle_Type", "Expected_Ride_Duration",
                                                      "Ride_Density", "Surge_Indicator", "Peak_Hours_Flag")]),
                     label = test_data$Historical_Cost_of_Ride)
params <- list(objective = "reg:squarederror", eval_metric = "rmse")
xgb_model <- xgboost(params = params, data = dtrain, nrounds = 100)
xgb_preds <- predict(xgb_model, dtest)
xgb_mae <- mean(abs(xgb_preds - test_data$Historical_Cost_of_Ride))
xgb_r2 <- cor(xgb_preds, test_data$Historical_Cost_of_Ride)^2

# Model performance comparison
cat("Random Forest MAE:", rf_mae, "R²:", rf_r2, "\n")
cat("XGBoost MAE:", xgb_mae, "R²:", xgb_r2, "\n")
install.packages("caret")
library(caret)
library(xgboost)

# Define all the predictor columns
predictor_columns <- c("Number_of_Riders", "Number_of_Drivers", "Location_Category",
                       "Customer_Loyalty_Status", "Number_of_Past_Rides",
                       "Average_Ratings", "Time_of_Booking", "Vehicle_Type",
                       "Expected_Ride_Duration",
                       "Ride_Density", "Surge_Indicator", "Peak_Hours_Flag")

# Create training data matrix
train_matrix <- xgb.DMatrix(data = as.matrix(train_data[, predictor_columns]), label = train_data$Historical_Cost_of_Ride)
test_matrix <- xgb.DMatrix(data = as.matrix(test_data[, predictor_columns]), label = test_data$Historical_Cost_of_Ride)

# Set up the caret training grid
xgb_grid <- expand.grid(
  nrounds = c(50, 100),         # number of boosting rounds
  max_depth = c(3, 6, 9),       # depth of trees
  eta = c(0.1, 0.3),            # learning rate
  gamma = 0,                    # minimum loss reduction
  colsample_bytree = 1,         # subsample ratio of columns
  min_child_weight = 1,         # minimum sum of instance weight
  subsample = 1                 # subsample ratio of training instance
)

# Train control
xgb_control <- trainControl(method = "cv", number = 5, verboseIter = FALSE)

# Train the model
set.seed(123)
xgb_tuned <- train(
  x = train_data[, predictor_columns],
  y = train_data$Historical_Cost_of_Ride,
  method = "xgbTree",
  trControl = xgb_control,
  tuneGrid = xgb_grid,
  verbose = FALSE
)

# Evaluate on test set
xgb_best_model <- xgb_tuned$finalModel
xgb_preds <- predict(xgb_tuned, newdata = test_data[, predictor_columns])
xgb_mae <- mean(abs(xgb_preds - test_data$Historical_Cost_of_Ride))
xgb_r2 <- cor(xgb_preds, test_data$Historical_Cost_of_Ride)^2

cat("Tuned XGBoost MAE:", xgb_mae, "R²:", xgb_r2, "\n")

# Save the best model
saveRDS(xgb_best_model, file = "best_model.rds")
# Define predictor columns
predictor_columns <- c("Number_of_Riders", "Number_of_Drivers", "Location_Category",
                       "Customer_Loyalty_Status", "Number_of_Past_Rides", "Average_Ratings",
                       "Time_of_Booking", "Vehicle_Type", "Expected_Ride_Duration",
                       "Ride_Density", "Surge_Indicator", "Peak_Hours_Flag")

# Set up tuning grid for Random Forest
rf_grid <- expand.grid(mtry = c(2, 4, 6, 8))  # number of variables randomly sampled at each split

# Set up train control for cross-validation
rf_control <- trainControl(method = "cv", number = 5, verboseIter = FALSE)

# Train the Random Forest model
set.seed(123)
rf_tuned <- train(
  x = train_data[, predictor_columns],
  y = train_data$Historical_Cost_of_Ride,
  method = "rf",
  trControl = rf_control,
  tuneGrid = rf_grid,
  ntree = 100,
  importance = TRUE
)

# Evaluate on test set
rf_preds <- predict(rf_tuned, newdata = test_data[, predictor_columns])
rf_mae <- mean(abs(rf_preds - test_data$Historical_Cost_of_Ride))
rf_r2 <- cor(rf_preds, test_data$Historical_Cost_of_Ride)^2

cat("Tuned Random Forest MAE:", rf_mae, "R²:", rf_r2, "\n")
# Model comparison summary
cat("Model Comparison Summary:\n")
cat("-------------------------------------------------\n")
cat("Previous Models:\n")
cat("Random Forest - MAE:", 55.93038, "R²:", 0.8754674, "\n")
cat("XGBoost (Before Tuning) - MAE:", 57.23552, "R²:", 0.8515301, "\n")
cat("-------------------------------------------------\n")
cat("Tuned Models:\n")
cat("Tuned Random Forest - MAE:", 51.7977, "R²:", 0.8793644, "\n")
cat("Tuned XGBoost - MAE:", 51.78446, "R²:", 0.884614, "\n")
cat("-------------------------------------------------\n")
cat("Conclusion: The tuned XGBoost model performs the best with the lowest MAE and highest R²,\n")
cat("closely followed by the tuned Random Forest. Tuning significantly improved both models' performance.\n")
install.packages("car")

# Feature importance
importance_matrix <- xgb.importance(model = xgb_best_model)
print(importance_matrix)
xgb.plot.importance(importance_matrix, top_n = 10)
library(ggplot2)
library(dplyr)

# Use data.table syntax directly
importance_matrix[order(-Frequency)][1:3]

# Additional insights
cat("\n✅ Insight:\n")
cat("- 'Expected_Ride_Duration' dominates all three metrics (Gain, Cover, and Frequency).\n")
cat("- 'Vehicle_Type' contributes significantly after that, especially in cover and frequency.\n")
cat("- 'Location_Category' and 'Time_of_Booking' have minimal impact.\n")

# Prepare the importance matrix as a tibble
importance_df <- as_tibble(importance_matrix) %>%
  slice_max(Frequency, n = 10) %>%
  arrange(desc(Frequency)) %>%
  mutate(Percentage = Frequency / sum(Frequency) * 100)

# Recalculate ypos in *cumulative order* (top of pie = top row of data)
importance_df <- importance_df %>%
  arrange(Percentage) %>%
  mutate(ypos = cumsum(Percentage) - 0.5 * Percentage)

# Plot donut chart
ggplot(importance_df, aes(x = 2, y = Percentage, fill = Feature)) +
  geom_bar(stat = "identity", width = 1, color = "white") +
  coord_polar("y", start = 0) +
  geom_text(aes(y = ypos, label = paste0(round(Percentage, 1), "%")),
            color = "white", size = 5) +
  theme_void() +
  xlim(0.5, 2.5) +
  ggtitle("Top 10 Feature Importances (Donut Chart)") +
  theme(legend.position = "right")
library(ggplot2)
library(dplyr)
library(scales)

# Prepare the data
importance_df <- as_tibble(importance_matrix) %>%
  slice_max(Frequency, n = 10) %>%
  arrange(desc(Frequency)) %>%
  mutate(
    Feature = factor(Feature, levels = Feature),  # enforce ordering for color mapping
    Percentage = Frequency / sum(Frequency) * 100,
    Label = paste0(Feature, ": ", round(Percentage, 1), "%")
  )

# Plot with correctly mapped legend colors
ggplot(importance_df, aes(x = "", y = Percentage, fill = Feature)) +
  geom_bar(stat = "identity", width = 1, color = "white") +
  coord_polar("y", start = 0) +
  scale_fill_brewer(palette = "Paired", labels = importance_df$Label) +
  theme_void() +
  labs(title = "Top 10 Feature Importances (Pie Chart)", fill = "Feature\n(Percentage)") +
  theme(
    plot.title = element_text(size = 20, face = "bold", hjust = 0.5),
    legend.position = "right",
    legend.title = element_text(size = 18, face = "bold"),
    legend.text = element_text(size = 20)
  )
# Actual vs Predicted Plot for XGBoost
library(ggplot2)

# Create a dataframe for plotting
comparison_df <- data.frame(
  Actual = test_data$Historical_Cost_of_Ride,
  Predicted = xgb_preds
)

# Poster-quality Actual vs Predicted Plot
ggplot(comparison_df, aes(x = Actual, y = Predicted)) +
  geom_point(color = "dodgerblue", alpha = 0.7, size = 2.5) +
  geom_abline(intercept = 0, slope = 1, linetype = "dashed", color = "darkred", size = 1) +
  labs(title = "Actual vs Predicted Ride Costs (XGBoost)",
       x = "Actual Ride Cost",
       y = "Predicted Ride Cost") +
  theme_minimal(base_size = 16) +
  theme(
    plot.title = element_text(hjust = 0.5, size = 20, face = "bold"),
    axis.title.x = element_text(size = 18, face = "bold"),
    axis.title.y = element_text(size = 18, face = "bold"),
    axis.text.x = element_text(size = 16),
    axis.text.y = element_text(size = 16),
    axis.line = element_line(size = 1.2, colour = "black"),
    panel.grid.major = element_line(color = "gray90"),
    panel.grid.minor = element_blank()
  ) +
  coord_equal()

