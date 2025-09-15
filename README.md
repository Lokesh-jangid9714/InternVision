Main Activity.java

package com.example.InternVision;

import android.content.Intent;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.ArrayAdapter;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;

import com.chaquo.python.PyObject;
import com.chaquo.python.Python;
import com.chaquo.python.android.AndroidPlatform;
import com.example.atry.adapter.InternshipAdapter;
import com.example.atry.databinding.ActivityMainBinding;
import com.example.atry.model.Internship;
import com.google.firebase.auth.FirebaseAuth;
import com.google.firebase.auth.FirebaseUser;
import com.google.gson.Gson;
import com.google.gson.reflect.TypeToken;

import java.lang.reflect.Type;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class MainActivity extends AppCompatActivity {

    private ActivityMainBinding binding;
    private InternshipAdapter adapter;
    private FirebaseAuth mAuth;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());

        mAuth = FirebaseAuth.getInstance();
        FirebaseUser currentUser = mAuth.getCurrentUser();

        if (currentUser == null) {
            // User login nahi hai, LoginActivity par bhejo
            startActivity(new Intent(MainActivity.this, LoginActivity.class));
            finish();
            return;
        }

        initPython();
        setupRecyclerView();
        setupUIListeners();

        // User ke email ke saath welcome message set karo
        binding.welcomeText.setText("Welcome, " + currentUser.getEmail());

        updateUIForLanguage("en");
    }

    private void initPython() {
        if (!Python.isStarted()) {
            Python.start(new AndroidPlatform(this));
        }
    }

    private void setupRecyclerView() {
        adapter = new InternshipAdapter();
        binding.recyclerView.setLayoutManager(new LinearLayoutManager(this));
        binding.recyclerView.setAdapter(adapter);
    }

    private void setupUIListeners() {
        binding.languageToggle.addOnButtonCheckedListener((group, checkedId, isChecked) -> {
            if (isChecked) {
                if (checkedId == R.id.englishButton) {
                    updateUIForLanguage("en");
                } else if (checkedId == R.id.gujaratiButton) {
                    updateUIForLanguage("gu");
                }
            }
        });

        binding.recommendButton.setOnClickListener(v -> fetchRecommendationsFromPython());

        // Logout button ka logic
        binding.logoutButton.setOnClickListener(v -> {
            mAuth.signOut();
            startActivity(new Intent(MainActivity.this, LoginActivity.class));
            finish();
        });
    }

    private void updateUIForLanguage(String lang) {
        binding.sectorDropdown.setText("", false);
        binding.educationDropdown.setText("", false);

        ArrayAdapter<CharSequence> sectorAdapter;
        ArrayAdapter<CharSequence> educationAdapter;

        if (lang.equals("gu")) {
            binding.titleText.setText(R.string.title_gu);
            binding.sectorLayout.setHint(getString(R.string.sector_hint_gu));
            binding.educationLayout.setHint(getString(R.string.education_hint_gu));
            binding.locationLayout.setHint(getString(R.string.location_hint_gu));
            binding.skillsLayout.setHint(getString(R.string.skills_hint_gu));
            binding.recommendButton.setText(R.string.recommend_button_gu);
            binding.emptyText.setText(R.string.empty_view_gu);
            sectorAdapter = ArrayAdapter.createFromResource(this, R.array.sector_items_gu, android.R.layout.simple_spinner_item);
            educationAdapter = ArrayAdapter.createFromResource(this, R.array.education_items_gu, android.R.layout.simple_spinner_item);
        } else {
            binding.titleText.setText(R.string.title_en);
            binding.sectorLayout.setHint(getString(R.string.sector_hint_en));
            binding.educationLayout.setHint(getString(R.string.education_hint_en));
            binding.locationLayout.setHint(getString(R.string.location_hint_en));
            binding.skillsLayout.setHint(getString(R.string.skills_hint_en));
            binding.recommendButton.setText(R.string.recommend_button_en);
            binding.emptyText.setText(R.string.empty_view_en);
            sectorAdapter = ArrayAdapter.createFromResource(this, R.array.sector_items_en, android.R.layout.simple_spinner_item);
            educationAdapter = ArrayAdapter.createFromResource(this, R.array.education_items_en, android.R.layout.simple_spinner_item);
        }
        binding.sectorDropdown.setAdapter(sectorAdapter);
        binding.educationDropdown.setAdapter(educationAdapter);
    }

    private void fetchRecommendationsFromPython() {
        String sector = binding.sectorDropdown.getText().toString();
        String education = binding.educationDropdown.getText().toString();
        String location = binding.locationInput.getText().toString();
        String skillsInput = binding.skillsInput.getText().toString().trim();

        if (skillsInput.isEmpty()) {
            Toast.makeText(this, "Please enter at least one skill", Toast.LENGTH_SHORT).show();
            return;
        }
        List<String> skillsList = new ArrayList<>(Arrays.asList(skillsInput.split("\\s*,\\s*")));
        showLoading(true);

        new Thread(() -> {
            String resultJson = "";
            try {
                Map<String, Object> allInputs = new HashMap<>();
                allInputs.put("sector", sector);
                allInputs.put("education", education);
                allInputs.put("location", location);
                allInputs.put("skills", skillsList);
                String inputsJsonString = new Gson().toJson(allInputs);

                Python py = Python.getInstance();
                PyObject recommenderModule = py.getModule("recommender");
                PyObject resultObject = recommenderModule.callAttr("recommend", inputsJsonString);
                resultJson = resultObject.toString();

                Type listType = new TypeToken<ArrayList<Internship>>() {}.getType();
                List<Internship> recommendedInternships = new Gson().fromJson(resultJson, listType);

                if (recommendedInternships != null && !recommendedInternships.isEmpty() && recommendedInternships.get(0).getTitle() == null && resultJson.contains("error")) {
                    Type errorType = new TypeToken<ArrayList<HashMap<String, String>>>(){}.getType();
                    List<HashMap<String, String>> errorList = new Gson().fromJson(resultJson, errorType);
                    String pythonError = errorList.get(0).get("error");
                    throw new Exception("Python Error: " + pythonError);
                }

                runOnUiThread(() -> {
                    showLoading(false);
                    if (recommendedInternships != null && !recommendedInternships.isEmpty()) {
                        adapter.updateData(recommendedInternships);
                        binding.recyclerView.setVisibility(View.VISIBLE);
                        binding.emptyText.setVisibility(View.GONE);
                    } else {
                        showEmptyView();
                    }
                });

            } catch (Exception e) {
                Log.e("ChaquopyError", "Error calling Python script", e);
                final String errorMessage = e.getMessage();
                runOnUiThread(() -> {
                    showLoading(false);
                    showEmptyView();
                    Toast.makeText(MainActivity.this, errorMessage, Toast.LENGTH_LONG).show();
                });
            }
        }).start();
    }

    private void showLoading(boolean isLoading) {
        binding.progressBar.setVisibility(isLoading ? View.VISIBLE : View.GONE);
        if (isLoading) {
            binding.recyclerView.setVisibility(View.GONE);
            binding.emptyText.setVisibility(View.GONE);
        }
    }

    private void showEmptyView() {
        binding.recyclerView.setVisibility(View.GONE);
        binding.emptyText.setVisibility(View.VISIBLE);
        adapter.updateData(null);
    }
}

