# COMPILER-DESIGN-BASICS
#include <iostream>
#include <string>
#include <vector>
#include <map>
#include <cctype>
#include <cmath>
#include <stdexcept>
#include <sstream>

class ExpressionParser {
private:
    std::string expr;
    size_t pos;
    std::map<std::string, double> variables;

    // Skip whitespace
    void skipWhitespace() {
        while (pos < expr.length() && std::isspace(expr[pos])) {
            pos++;
        }
    }

    // Parse a number or variable
    double parseNumber() {
        skipWhitespace();
        
        // Check for negative number
        bool isNegative = false;
        if (pos < expr.length() && expr[pos] == '-') {
            isNegative = true;
            pos++;
            skipWhitespace();
        }
        
        // Handle parentheses
        if (pos < expr.length() && expr[pos] == '(') {
            pos++; // Skip opening parenthesis
            double result = parseExpression();
            skipWhitespace();
            
            if (pos >= expr.length() || expr[pos] != ')') {
                throw std::runtime_error("Missing closing parenthesis");
            }
            pos++; // Skip closing parenthesis
            return isNegative ? -result : result;
        }
        
        // Handle functions
        if (pos < expr.length() && std::isalpha(expr[pos])) {
            std::string identifier;
            while (pos < expr.length() && (std::isalnum(expr[pos]) || expr[pos] == '_')) {
                identifier += expr[pos++];
            }
            skipWhitespace();
            
            // Check if it's a function
            if (pos < expr.length() && expr[pos] == '(') {
                pos++; // Skip opening parenthesis
                double argument = parseExpression();
                skipWhitespace();
                
                if (pos >= expr.length() || expr[pos] != ')') {
                    throw std::runtime_error("Missing closing parenthesis in function call");
                }
                pos++; // Skip closing parenthesis
                
                double result;
                if (identifier == "sin") {
                    result = std::sin(argument);
                } else if (identifier == "cos") {
                    result = std::cos(argument);
                } else if (identifier == "tan") {
                    result = std::tan(argument);
                } else if (identifier == "sqrt") {
                    if (argument < 0) {
                        throw std::runtime_error("Cannot take square root of negative number");
                    }
                    result = std::sqrt(argument);
                } else if (identifier == "log") {
                    if (argument <= 0) {
                        throw std::runtime_error("Cannot take logarithm of non-positive number");
                    }
                    result = std::log(argument);
                } else if (identifier == "exp") {
                    result = std::exp(argument);
                } else if (identifier == "abs") {
                    result = std::abs(argument);
                } else {
                    throw std::runtime_error("Unknown function: " + identifier);
                }
                
                return isNegative ? -result : result;
            }
            
            // It's a variable
            if (variables.find(identifier) == variables.end()) {
                throw std::runtime_error("Undefined variable: " + identifier);
            }
            
            return isNegative ? -variables[identifier] : variables[identifier];
        }
        
        // It's a number
        std::string numStr;
        while (pos < expr.length() && (std::isdigit(expr[pos]) || expr[pos] == '.')) {
            numStr += expr[pos++];
        }
        
        if (numStr.empty()) {
            throw std::runtime_error("Expected number");
        }
        
        double value;
        std::istringstream iss(numStr);
        if (!(iss >> value)) {
            throw std::runtime_error("Invalid number format: " + numStr);
        }
        
        return isNegative ? -value : value;
    }

    // Parse multiplication and division
    double parseTerm() {
        double left = parseNumber();
        
        while (true) {
            skipWhitespace();
            
            if (pos >= expr.length()) {
                break;
            }
            
            char op = expr[pos];
            if (op != '*' && op != '/' && op != '%') {
                break;
            }
            
            pos++; // Skip operator
            double right = parseNumber();
            
            if (op == '*') {
                left *= right;
            } else if (op == '/') {
                if (right == 0) {
                    throw std::runtime_error("Division by zero");
                }
                left /= right;
            } else if (op == '%') {
                if (right == 0) {
                    throw std::runtime_error("Modulo by zero");
                }
                left = std::fmod(left, right);
            }
        }
        
        return left;
    }

    // Parse addition and subtraction
    double parseExpression() {
        double left = parseTerm();
        
        while (true) {
            skipWhitespace();
            
            if (pos >= expr.length()) {
                break;
            }
            
            char op = expr[pos];
            if (op != '+' && op != '-') {
                break;
            }
            
            pos++; // Skip operator
            double right = parseTerm();
            
            if (op == '+') {
                left += right;
            } else {
                left -= right;
            }
        }
        
        return left;
    }

    // Parse an assignment
    void parseAssignment() {
        skipWhitespace();
        
        if (pos >= expr.length() || !std::isalpha(expr[pos])) {
            throw std::runtime_error("Variable name must start with a letter");
        }
        
        std::string varName;
        while (pos < expr.length() && (std::isalnum(expr[pos]) || expr[pos] == '_')) {
            varName += expr[pos++];
        }
        
        skipWhitespace();
        
        if (pos >= expr.length() || expr[pos] != '=') {
            throw std::runtime_error("Expected '=' in assignment");
        }
        
        pos++; // Skip '='
        double value = parseExpression();
        variables[varName] = value;
        
        std::cout << varName << " = " << value << std::endl;
    }

public:
    ExpressionParser() : pos(0) {
        // Initialize with some mathematical constants
        variables["pi"] = M_PI;
        variables["e"] = M_E;
    }

    double evaluate(const std::string& expression) {
        expr = expression;
        pos = 0;
        
        skipWhitespace();
        
        // Check for assignment
        size_t equalsPos = expression.find('=');
        if (equalsPos != std::string::npos) {
            // Check if it's a valid assignment (variable on left side)
            bool validAssignment = true;
            for (size_t i = 0; i < equalsPos; i++) {
                if (!std::isspace(expression[i]) && !std::isalnum(expression[i]) && expression[i] != '_') {
                    validAssignment = false;
                    break;
                }
            }
            
            if (validAssignment) {
                parseAssignment();
                return variables[expression.substr(0, equalsPos)];
            }
        }
        
        double result = parseExpression();
        skipWhitespace();
        
        if (pos < expr.length()) {
            throw std::runtime_error("Unexpected character: " + std::string(1, expr[pos]));
        }
        
        return result;
    }

    void showVariables() const {
        std::cout << "Variables:" << std::endl;
        for (const auto& var : variables) {
            std::cout << var.first << " = " << var.second << std::endl;
        }
    }

    void clearVariables() {
        variables.clear();
        // Reinitialize constants
        variables["pi"] = M_PI;
        variables["e"] = M_E;
    }
};

int main() {
    ExpressionParser parser;
    std::string input;
    
    std::cout << "Expression Parser" << std::endl;
    std::cout << "=================" << std::endl;
    std::cout << "Type 'exit' to quit, 'vars' to show variables, 'clear' to clear variables" << std::endl;
    std::cout << "Supported operations: +, -, *, /, %, (, )" << std::endl;
    std::cout << "Supported functions: sin, cos, tan, sqrt, log, exp, abs" << std::endl;
    std::cout << "Built-in constants: pi, e" << std::endl;
    std::cout << std::endl;
    
    while (true) {
        std::cout << "> ";
        std::getline(std::cin, input);
        
        if (input == "exit" || input == "quit") {
            break;
        } else if (input == "vars") {
            parser.showVariables();
        } else if (input == "clear") {
            parser.clearVariables();
            std::cout << "Variables cleared." << std::endl;
        } else if (!input.empty()) {
            try {
                double result = parser.evaluate(input);
                
                // Only print the result if it's not an assignment
                if (input.find('=') == std::string::npos) {
                    std::cout << "= " << result << std::endl;
                }
            } catch (const std::exception& e) {
                std::cout << "Error: " << e.what() << std::endl;
            }
        }
    }
    
    return 0;
}
